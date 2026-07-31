---
read_when:
    - De Android-Node koppelen of opnieuw verbinden
    - Problemen met Gateway-detectie of authenticatie op Android oplossen
    - Een Android-apparaat spiegelen of bedienen vanaf een externe Mac
    - Pariteit van de chatgeschiedenis tussen clients verifiëren
summary: 'Android-app (node): verbindingsdraaiboek + opdrachtoppervlak voor Verbinden/Chat/OpenClaw/Spraak/Canvas'
title: Android-app
x-i18n:
    generated_at: "2026-07-27T05:38:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a134a678e26924abc24dd107c3feaad9d09e83e3829eef73514c7ef078d578f1
    source_path: platforms/android.md
    workflow: 16
---

<Note>
De officiële Android-app is beschikbaar op [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) en als ondertekende zelfstandige APK in ondersteunde [GitHub-releases](https://github.com/openclaw/openclaw/releases). Het is een begeleidende Node en vereist een actieve OpenClaw Gateway. Bron: [apps/android](https://github.com/openclaw/openclaw/tree/main/apps/android) ([bouwinstructies](https://github.com/openclaw/openclaw/blob/main/apps/android/README.md)).
</Note>

## Ondersteuningsoverzicht

- Rol: app voor een begeleidende Node (Android host de Gateway niet).
- Gateway vereist: ja (voer deze uit op macOS, Linux of Windows via WSL2).
- Installatie: [Google Play](https://play.google.com/store/apps/details?id=ai.openclaw.app&hl=en_IN) of `OpenClaw-Android.apk` uit een ondersteunde [GitHub-release](https://github.com/openclaw/openclaw/releases), [Aan de slag](/nl/start/getting-started) voor de Gateway en vervolgens [Koppelen](/nl/channels/pairing).
- Gateway: [Draaiboek](/nl/gateway) + [Configuratie](/nl/gateway/configuration).
  - Protocollen: [Gateway-protocol](/nl/gateway/protocol) (Nodes + besturingsvlak).
- **Settings → OpenClaw** opent een speciale assistent voor Gateway-instellingen wanneer de operatorverbinding `operator.admin` heeft en de Gateway `openclaw.chat` ondersteunt. Het installatiegesprek blijft gescheiden van gewone chats, maskeert geheime antwoorden lokaal en schakelt pas over naar de chat nadat je op **Open Chat** tikt.

Systeembesturing (launchd/systemd) vindt plaats op de Gateway-host — zie [Gateway](/nl/gateway).

## Gelijktijdige Gateway-sessies

Koppel elke Gateway één keer en open vervolgens **Settings → Gateway**. Het vinkje markeert
de Gateway waarop de focus ligt en elke schakelaar bepaalt of de operatorsessie van een Gateway
waarop de focus niet ligt verbonden blijft. Ingeschakelde Gateways maken onafhankelijk opnieuw
verbinding terwijl de app op de voorgrond staat, zodat het wisselen van focus de andere
verbindingen niet verbreekt. Alleen de Gateway waarop de focus ligt, beheert de Android-Node-sessie
en apparaatmogelijkheden; dit voorkomt dat meerdere Gateways tegelijk camera-, locatie-, scherm-
of meldingsopdrachten naar dezelfde telefoon sturen. Android kan de secundaire verbindingen
opschorten nadat de app de voorgrond verlaat.

## Wear OS-begeleidende app

De begeleidende Wear OS-app gebruikt de geverifieerde Gateway-verbinding van de gekoppelde Android-telefoon; het horloge ontvangt of bewaart nooit Gateway-inloggegevens. De app kan agents en sessies selecteren, begrensde transcripties lezen, tekst of gedicteerde antwoorden verzenden, een actieve uitvoering afbreken, realtime Talk starten binnen de geselecteerde sessie en de Gateway van de gekoppelde telefoon verbinden of loskoppelen. De app biedt ook lokale antwoordmeldingen, een donkere of lichte weergave en optionele automatische spraak voor antwoorden. Over de mogelijkheden van agent- en Gateway-besturing wordt onderhandeld om gespreide updates van telefoon en horloge te ondersteunen. Realtime Talk streamt microfoon- en afspeelaudio via een tijdelijk Wear OS Data Layer-kanaal en stopt wanneer de geselecteerde telefoon, Gateway-verbinding of het audiokanaal verloren gaat.

## Installeren buiten Google Play

Reguliere definitieve en correctie-GitHub-releases bevatten een universele `OpenClaw-Android.apk` en `OpenClaw-Android-SHA256SUMS.txt`. De APK wordt gebouwd op basis van de releasetag, ondertekend met de OpenClaw Android-releasesleutel en bevat herkomstinformatie van GitHub Actions.

Kies een [release](https://github.com/openclaw/openclaw/releases) waarin beide assets worden vermeld en download en verifieer vervolgens exact die tag voordat je de app sideloadt:

```bash
release_tag=vYYYY.M.PATCH
gh release download "$release_tag" \
  --repo openclaw/openclaw \
  --pattern OpenClaw-Android.apk \
  --pattern OpenClaw-Android-SHA256SUMS.txt
sha256sum --check OpenClaw-Android-SHA256SUMS.txt
gh attestation verify OpenClaw-Android.apk \
  --repo openclaw/openclaw \
  --signer-workflow openclaw/openclaw/.github/workflows/android-release.yml \
  --source-ref "refs/tags/${release_tag}" \
  --deny-self-hosted-runners
```

<Warning>
Installaties via Google Play en een zelfstandige APK gebruiken verschillende updatekanalen en kunnen verschillende ondertekeningsidentiteiten hebben. Android kan vereisen dat de bestaande app wordt verwijderd voordat je van kanaal wisselt, waardoor de lokale appgegevens worden verwijderd. Blijf voor normale updates bij één kanaal.
</Warning>

## Android spiegelen en besturen vanaf een externe Mac

[scrcpy](https://github.com/Genymobile/scrcpy) spiegelt een Android-scherm in een macOS-venster en
stuurt toetsenbord- en aanwijzerinvoer door via Android Debug Bridge (ADB). Dit is een workflow aan
de operatorzijde, los van de OpenClaw-Node-verbinding. Dit is nuttig wanneer het Android-apparaat en
de Mac zich op verschillende locaties bevinden, maar hetzelfde besloten Tailscale-netwerk delen.

### Voordat je begint

- Installeer Tailscale op het Android-apparaat en de Mac en verbind beide met dezelfde tailnet.
- Schakel op Android **Developer options** en **USB debugging** in. In Android 16 staat **Wireless
  debugging** onder **Settings > System > Developer options**. Zie [Android-ontwikkelaarsopties](https://developer.android.com/studio/debug/dev-options).
- Installeer scrcpy en ADB op de Mac:

  ```bash
  brew install scrcpy
  brew install --cask android-platform-tools
  ```

- Houd het Android-apparaat beschikbaar voor de eerste verbinding. Android moet de ADB-sleutel
  van elke Mac goedkeuren voordat die Mac het apparaat kan besturen.

### ADB via TCP inschakelen

Sluit voor de eerste configuratie het Android-apparaat via USB aan op een vertrouwde computer en
keur de foutopsporingsprompt goed. Voer vervolgens het volgende uit:

```bash
adb devices
adb tcpip 5555
```

Je kunt USB nu loskoppelen. Als poort 5555 na het opnieuw opstarten van een apparaat of het opnieuw
instellen van foutopsporing niet meer luistert, herhaal je deze lokale configuratiestap. Android 11
en nieuwer kunnen het initiële vertrouwen ook instellen met
**Wireless debugging > Pair device with pairing code** en `adb pair`.

### Alleen de besturende Mac toestaan

Tailnets met beperkende toekenningen moeten de besturende Mac expliciet toestaan TCP-poort 5555
op het Android-apparaat te bereiken. Voeg een beperkte regel toe aan het tailnetbeleid en vervang
de voorbeeldadressen door de stabiele Tailscale-IP-adressen van de twee apparaten:

```json5
{
  grants: [
    {
      src: ["<remote-mac-tailnet-ip>"],
      dst: ["<android-tailnet-ip>"],
      ip: ["tcp:5555"],
    },
  ],
}
```

Zie [Tailscale-toekenningen](https://tailscale.com/docs/reference/syntax/grants) voor hostaliassen en
andere selectors. Verleen de toegang tot deze poort niet aan het openbare internet en stel deze niet
beschikbaar met Funnel: een geautoriseerde ADB-client heeft vergaande controle over het apparaat.

### Verbinding maken en spiegelen starten

Op de externe Mac:

```bash
adb connect <android-tailnet-ip>:5555
adb devices
scrcpy --serial <android-tailnet-ip>:5555
```

De eerste `adb connect` vanaf deze Mac toont een autorisatiedialoogvenster op Android. Ontgrendel
het apparaat, bevestig de sleutelvingerafdruk en selecteer **Always allow from this computer** alleen
wanneer de Mac wordt vertrouwd. Een geslaagde `adb devices`-vermelding eindigt op
`device`; `unauthorized` betekent dat de prompt op het apparaat niet is goedgekeurd.

Zodra het scrcpy-venster wordt geopend, gebruik je het rechtstreeks of bestuur je het met een
macOS-tool voor schermautomatisering, zoals [Peekaboo](https://peekaboo.sh/). scrcpy verzorgt het
beeld en de invoer; Tailscale biedt alleen het besloten netwerkpad.

### Problemen oplossen

- `Connection timed out`: controleer de tailnettoekenning voor TCP 5555. Een geslaagde
  `tailscale ping` bewijst dat de peer bereikbaar is, niet dat het beleid deze TCP-poort toestaat.
  Test dit vanaf de Mac met `nc -vz <android-tailnet-ip> 5555`.
- `unauthorized`: ontgrendel Android en keur de ADB-sleutel van de externe Mac goed,
  of verwijder het verouderde werkstation onder **Wireless debugging > Paired devices** en koppel het opnieuw.
- `Connection refused`: maak lokaal opnieuw verbinding en voer `adb tcpip 5555` opnieuw uit.
- Meer dan één apparaat vermeld: behoud het expliciete argument `--serial <android-tailnet-ip>:5555`.

Sluit scrcpy en verbreek de ADB-verbinding wanneer je klaar bent:

```bash
adb disconnect <android-tailnet-ip>:5555
```

## Verbindingsdraaiboek

Android-Node-app ⇄ (mDNS/NSD + WebSocket) ⇄ **Gateway**

Android maakt rechtstreeks verbinding met de Gateway-WebSocket en gebruikt apparaatkoppeling (`role: node`).

Voor Tailscale- of openbare hosts vereist Android een beveiligd eindpunt:

- Voorkeur: Tailscale Serve / Funnel met `https://<magicdns>` / `wss://<magicdns>`
- Ook ondersteund: elke andere `wss://`-Gateway-URL met een echt TLS-eindpunt
- Niet-versleutelde `ws://` blijft ondersteund op adressen van besloten LAN's / `.local`-hosts, plus `localhost`, `127.0.0.1` en de Android-emulatorbridge (`10.0.2.2`); bij configuratie buiten loopback wordt automatisch beperkte operatortoegang gebruikt

### Vereisten

- Gateway actief op een andere machine (of bereikbaar via SSH).
- Android-apparaat/-emulator kan de Gateway-WebSocket bereiken:
  - Hetzelfde LAN met mDNS/NSD, **of**
  - Dezelfde Tailscale-tailnet met Wide-Area Bonjour / unicast DNS-SD (zie hieronder), **of**
  - Handmatige Gateway-host/-poort (fallback)
- Mobiele koppeling via tailnet/openbaar netwerk gebruikt **geen** onbewerkte tailnet-IP-`ws://`-eindpunten. Gebruik in plaats daarvan Tailscale Serve of een andere `wss://`-URL.
- De CLI `openclaw` is beschikbaar op de Gateway-machine (of via SSH) om koppelingsverzoeken goed te keuren.

### 1. Start de Gateway

```bash
openclaw gateway --port 18789 --verbose
```

Controleer of je in de logboeken iets als het volgende ziet:

- `listening on ws://0.0.0.0:18789`

Geef voor externe Android-toegang via Tailscale de voorkeur aan Serve/Funnel boven een rechtstreekse tailnetbinding:

```bash
openclaw gateway --tailscale serve
```

Dit biedt Android een beveiligd `wss://`- / `https://`-eindpunt. Een gewone
`gateway.bind: "tailnet"`-configuratie is niet voldoende voor de eerste externe Android-koppeling, tenzij
je TLS ook afzonderlijk beëindigt.

### 2. Detectie verifiëren (optioneel)

Vanaf de Gateway-machine:

```bash
dns-sd -B _openclaw-gw._tcp local.
```

Meer opmerkingen over foutopsporing: [Bonjour](/nl/gateway/bonjour).

Als je ook een detectiedomein voor een groot gebied hebt geconfigureerd, vergelijk dit dan met:

```bash
openclaw gateway discover --json
```

Dit toont in één keer `local.` plus het geconfigureerde domein voor een groot gebied, waarbij
het opgeloste service-eindpunt wordt gebruikt in plaats van alleen TXT-hints.

#### Netwerkoverschrijdende detectie via unicast DNS-SD

Android NSD/mDNS-detectie werkt niet over meerdere netwerken heen. Als de Android-Node en de Gateway
zich op verschillende netwerken bevinden, maar via Tailscale zijn verbonden, gebruik dan Wide-Area
Bonjour / unicast DNS-SD. Alleen detectie is niet voldoende voor Android-koppeling via een
tailnet/openbaar netwerk — de gedetecteerde route heeft nog steeds een beveiligd eindpunt nodig
(`wss://` of Tailscale Serve):

1. Stel een DNS-SD-zone in (bijvoorbeeld `openclaw.internal.`) op de Gateway-host en publiceer `_openclaw-gw._tcp`-records.
2. Configureer Tailscale split DNS voor het gekozen domein en laat dit naar die DNS-server verwijzen.

Details en een voorbeeldconfiguratie voor CoreDNS: [Bonjour](/nl/gateway/bonjour).

### 3. Verbinding maken vanaf Android

In de Android-app:

- De app houdt de Gateway-verbinding actief via een **voorgrondservice** (permanente melding).
- Open het tabblad **Connect**.
- Gebruik de modus **Setup Code** of **Manual**.
- Als detectie wordt geblokkeerd, gebruik je handmatig de host/poort in **Advanced controls**. Voor hosts op een besloten LAN werkt `ws://` nog steeds. Schakel voor Tailscale-/openbare hosts TLS in en gebruik een `wss://`- / Tailscale Serve-eindpunt.

Na de eerste geslaagde koppeling maakt Android bij het starten automatisch opnieuw verbinding met de actieve gekoppelde Gateway (op basis van best effort voor gedetecteerde Gateways, die zichtbaar moeten zijn op het netwerk).

Officiële installatiecodes verbinden Android als een knooppunt en verlenen standaard volledige operatortoegang tot de Gateway via `wss://`. Installatie via `ws://` met niet-loopback-plaintext gebruikt automatisch beperkte toegang voor de veiligheid van bearer-tokens. **Instellingen → Gateway** toont **Volledige** of **Beperkte** toegang. Configureer voor een beperkte verbinding `wss://` of Tailscale Serve, genereer een nieuwe code met volledige toegang in de Control UI of met `openclaw qr`, scan of plak deze vervolgens op die pagina en maak opnieuw verbinding. Operators die het beperkte profiel willen, kunnen **Beperkte toegang** selecteren in de Control UI of `openclaw qr --limited` uitvoeren.

### Gekoppelde Gateways beheren

De app houdt een register bij van elke Gateway waarmee deze is gekoppeld, zodat je operatorsessies verbonden kunt houden en de focus kunt wijzigen zonder opnieuw te koppelen:

- **Instellingen → Gateway** vermeldt gekoppelde Gateways en markeert de Gateway die de focus heeft. Tik op een vermelding om deze de focus te geven; de andere ingeschakelde operatorsessies blijven verbonden.
- Elke schakelaar bepaalt of de Gateway die niet de focus heeft verbonden blijft terwijl de app op de voorgrond staat. De Gateway met de focus blijft ingeschakeld en beheert de knooppuntverbinding en apparaatmogelijkheden van de telefoon.
- Het tabblad **Verbinden** toont een snelle wisselaar wanneer meer dan één Gateway is gekoppeld.
- Inloggegevens, apparaattokens, TLS-vertrouwen, chatgeschiedenis en in de wachtrij geplaatste offlineberichten worden per Gateway opgeslagen. Bij het wijzigen van de focus wordt de status van Gateways nooit vermengd en offline in de wachtrij geplaatste berichten worden uitsluitend afgeleverd bij de Gateway waarvoor ze zijn geschreven.
- **Vergeten** verwijdert de registervermelding van een Gateway, samen met de bijbehorende inloggegevens, apparaattokens, TLS-pin en gecachte chats.

### Aanwezigheidsbakens voor activiteit

Nadat de geverifieerde knooppuntsessie verbinding heeft gemaakt, en wanneer de app naar de achtergrond gaat terwijl de voorgrondservice nog verbonden is, roept Android `node.event` aan met `event: "node.presence.alive"`. De Gateway registreert dit als `lastSeenAtMs`/`lastSeenReason` in de metadata van het gekoppelde knooppunt/apparaat, maar pas nadat de identiteit van het geverifieerde knooppuntapparaat bekend is.

De app beschouwt het baken alleen als succesvol geregistreerd wanneer het antwoord van de Gateway `handled: true` bevat. Oudere Gateways kunnen `node.event` bevestigen met `{ "ok": true }`; dat antwoord is compatibel, maar geldt niet als een duurzame update van het tijdstip waarop het apparaat voor het laatst is gezien.

### 4. Koppeling goedkeuren (CLI)

Op de Gateway-machine:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Koppelingsdetails: [Koppeling](/nl/channels/pairing).

Optioneel: als het Android-knooppunt altijd verbinding maakt vanuit een streng gecontroleerd subnet, kun je automatische goedkeuring bij de eerste knooppuntkoppeling inschakelen met expliciete CIDR's of exacte IP-adressen:

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

Dit is standaard uitgeschakeld. Het is alleen van toepassing op een nieuwe `role: node`-koppeling zonder aangevraagde scopes. Koppeling van operators/browsers en elke wijziging van rol, scope, metadata of openbare sleutel vereisen nog steeds handmatige goedkeuring.

### 5. Controleren of het knooppunt verbonden is

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

### 6. Chat en geschiedenis

Het Android-tabblad Chat ondersteunt sessieselectie (standaard `main`, plus andere bestaande sessies):

- Geschiedenis: `chat.history` (genormaliseerd voor weergave — inline richtlijntags, XML-payloads van toolaanroepen in plaintext (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>` en afgekorte varianten) en gelekte ASCII-/volledige-breedtebesturingstokens van modellen worden verwijderd; assistentrijen met uitsluitend stille tokens, zoals exact `NO_REPLY` / `no_reply`, worden weggelaten; te grote rijen kunnen worden vervangen door tijdelijke aanduidingen)
- Verzenden: `chat.send`
- Duurzaam verzenden: elke verzending (tekst, geselecteerde afbeeldingen en spraaknotities) wordt vóór elke netwerkpoging vastgelegd in een outbox op het apparaat per Gateway, zodat het beëindigen van de app verzonden invoer niet kan laten verloren gaan. Verzendingen die offline in de wachtrij zijn geplaatst, worden na het opnieuw verbinden op volgorde afgeleverd met stabiele idempotentiesleutels. Een verzending wordt pas verwijderd nadat de beurt zichtbaar is in de canonieke `chat.history` — alleen een bevestiging geldt niet als bewijs van aflevering. Ambigue uitkomsten (verloren bevestiging, app beëindigd tijdens het verzenden, herstart van de Gateway voordat het transcript is geschreven) verschijnen als zichtbare rijen met expliciete opties **Opnieuw proberen**/**Verwijderen**, in plaats van automatisch opnieuw te verzenden. Slashopdrachten worden na opnieuw verbinden nooit automatisch herhaald; ze wachten op een expliciete nieuwe poging. De wachtrij is begrensd (50 berichten en 48 MB aan bijlagebytes per Gateway) en niet-verzonden rijen verlopen na 48 uur. Concepten in het invoerveld die nooit zijn verzonden, blijven niet bewaard wanneer het proces wordt beëindigd.
- Pushupdates (naar beste vermogen): `chat.subscribe` -> `event:"chat"`
- Luisteren: houd een assistentbericht ingedrukt en kies **Luisteren** om het te horen; audio wordt via Gateway-`tts.speak` gerenderd met de geconfigureerde keten van TTS-providers. TTS van het systeem op het apparaat wordt gebruikt wanneer de Gateway geen audio kan renderen. Het afspelen stopt bij het wisselen van sessie, een nieuwe chat, wanneer de app naar de achtergrond gaat of wanneer de chat wordt gesloten.

### 7. Canvas en camera

#### Gateway-canvas-host (aanbevolen voor webinhoud)

Als je wilt dat het knooppunt echte HTML/CSS/JS toont die de agent op schijf kan bewerken, richt je het knooppunt op de Gateway-canvas-host.

<Note>
Knooppunten laden het canvas vanaf de HTTP-server van de Gateway (dezelfde poort als `gateway.port`, standaard `18789`).
</Note>

1. Maak `~/.openclaw/workspace/canvas/index.html` op de Gateway-host.
2. Navigeer het knooppunt ernaartoe (LAN):

```bash
openclaw nodes invoke --node "<Android Node>" --command canvas.navigate --params '{"url":"http://<gateway-hostname>.local:18789/__openclaw__/canvas/"}'
```

Tailnet (optioneel): als beide apparaten Tailscale gebruiken, gebruik je een MagicDNS-naam of tailnet-IP in plaats van `.local`, bijvoorbeeld `http://<gateway-magicdns>:18789/__openclaw__/canvas/`.

Deze server injecteert een client voor automatisch opnieuw laden in HTML en laadt opnieuw bij bestandswijzigingen. De Gateway biedt ook `/__openclaw__/a2ui/` aan, maar de Android-app behandelt externe A2UI-pagina's uitsluitend als renderweergaven. A2UI-opdrachten met acties gebruiken de gebundelde A2UI-pagina die eigendom is van de app.

Canvasopdrachten (alleen op de voorgrond):

- `canvas.eval`, `canvas.snapshot`, `canvas.navigate` (gebruik `{"url":""}` of `{"url":"/"}` om terug te keren naar het standaardsjabloon). `canvas.snapshot` retourneert `{ format, base64 }` (standaard `format="jpeg"`).
- A2UI: `canvas.a2ui.push`, `canvas.a2ui.reset` (verouderde alias `canvas.a2ui.pushJSONL`). Deze gebruiken de gebundelde A2UI-pagina die eigendom is van de app voor rendering met acties.

Cameraopdrachten (alleen op de voorgrond; afhankelijk van toestemming): `camera.snap` (jpg), `camera.clip` (mp4). Zie [Cameraknooppunt](/nl/nodes/camera) voor parameters en CLI-hulpmiddelen.

### 8. Spraak en uitgebreid Android-opdrachtenoppervlak

- De hoofdnavigatie van Android bestaat uit **Start**, **Chat** en **Instellingen**. Spraakinvoer
  hoort bij het invoerveld van Chat; er is geen afzonderlijk tabblad Spraak.
- Tik op de microfoon van het invoerveld voor spraakherkenning op het apparaat, waarmee een
  transcript in het concept wordt ingevoegd. Houd de microfoon ingedrukt om een
  spraaknotitiebijlage op te nemen. De gebruikersinterface meldt niet-beschikbare herkenning, ontbrekende toestemming,
  bezet-/netwerkfouten en gevallen waarin geen spraak is gedetecteerd, in plaats van de
  poging stilzwijgend te negeren.
- Start continu **Praten** via de Chat-golfvorm. Dicteren, het opnemen van
  spraaknotities en Praten sluiten elkaar wederzijds uit als microfoonpaden.
- De Praatmodus promoveert de bestaande voorgrondservice van `connectedDevice` naar `connectedDevice|microphone` voordat het opnemen begint en degradeert deze weer wanneer de Praatmodus stopt. De knooppuntservice declareert `FOREGROUND_SERVICE_CONNECTED_DEVICE` met `CHANGE_NETWORK_STATE`; Android 14+ vereist ook de `FOREGROUND_SERVICE_MICROPHONE`-declaratie, de `RECORD_AUDIO`-runtimetoekenning en het microfoonservicetype tijdens runtime.
- Standaard gebruikt Android Praten systeemeigen spraakherkenning, Gateway-chat en `talk.speak` via de geconfigureerde Praatprovider van de Gateway. Lokale systeem-TTS wordt alleen gebruikt wanneer `talk.speak` niet beschikbaar is.
- Android Praten gebruikt alleen realtime Gateway-relay wanneer `talk.realtime.mode` gelijk is aan `realtime` en `talk.realtime.transport` gelijk is aan `gateway-relay`.
- Android maakt geen reclame voor de mogelijkheid `voiceWake`. Gebruik Chat-dicteren,
  een spraaknotitie of Praten voor spraakinvoer.
- Aanvullende Android-opdrachtfamilies (beschikbaarheid is afhankelijk van apparaat, toestemmingen en gebruikersinstellingen):
  - `device.status`, `device.info`, `device.permissions`, `device.health`
  - `device.apps` alleen wanneer **Instellingen > Telefoonmogelijkheden > Geïnstalleerde apps** is ingeschakeld; standaard worden apps vermeld die zichtbaar zijn in de launcher (geef `includeNonLaunchable` door voor de volledige lijst).
  - `notifications.list`, `notifications.actions` (zie [Meldingen doorsturen](#notification-forwarding) hieronder)
  - `photos.latest`
  - `contacts.search`, `contacts.add`
  - `calendar.events`, `calendar.add`
  - `callLog.search`
  - `sms.search`
  - `motion.activity`, `motion.pedometer`

### 9. Werkruimtebestanden (alleen-lezen)

Het Start-overzicht bevat een kaart **Bestanden** waarmee via de alleen-lezen Gateway-RPC's `agents.workspace.list` / `agents.workspace.get` door de werkruimte van de actieve agent kan worden gebladerd: navigeren door mappen, voorbeelden van tekst en afbeeldingen en exporteren via het Android-deelvenster. Er zijn geen schrijfbewerkingen en de Gateway begrenst de grootte van voorbeelden.

## Goedkeuringen van opdrachten beoordelen

Een operatorverbinding met `operator.admin`, of een gekoppelde
`operator.approvals`-verbinding die expliciet door de Gateway als doel is ingesteld, kan
openstaande uitvoeringsverzoeken beoordelen onder **Instellingen -> Goedkeuringen**. De app laadt de
opgeschoonde goedkeuringsrecord van de Gateway voordat de knoppen worden ingeschakeld, toont elke
beveiligingswaarschuwing en de exacte beslissingen die door dat verzoek worden aangeboden, en stuurt
de goedkeurings-ID en het eigenaartype terug naar de Gateway.

De goedkeuringsstatus wordt gedeeld met de Control UI en ondersteunde chatoppervlakken. Het
eerste vastgelegde antwoord wint; Android toont dat canonieke resultaat, zelfs wanneer
een ander oppervlak als eerste heeft geantwoord. Als een antwoord op het oplossen verloren gaat of de Gateway
de verbinding verbreekt, houdt de app de actie vergrendeld en leest deze de goedkeuring opnieuw
voordat een andere beslissing wordt aangeboden.

Gateways van vóór de uniforme goedkeuringsmethoden vallen terug op de meegeleverde
uitvoeringsspecifieke methoden. Beoordeling van openstaande verzoeken blijft werken, maar voor de bewaarde terminalstatus
en het uitgebreidere resultaat voor meerdere oppervlakken is een bijgewerkte Gateway vereist.

## Vragen van agents beantwoorden

Chat toont openstaande Gateway-vragen als systeemeigen kaarten voor operatorverbindingen
met `operator.questions` (of `operator.admin`). Kaarten ondersteunen enkelvoudige en
meervoudige selectieopties, optiebeschrijvingen, vrije-tekst-antwoorden via **Anders** en een
aftelling tot het verlopen. Na opnieuw verbinden worden openstaande vragen opnieuw geladen vanuit de Gateway. Een kaart
wordt vergrendeld wanneer dit apparaat deze beantwoordt, een ander oppervlak deze eerst beantwoordt, of de
vraag verloopt of wordt geannuleerd.

## Assistentingangen

Android ondersteunt het starten van OpenClaw via de systeemassistenttrigger (Google Assistant). Als je de startknop ingedrukt houdt (of een andere `ACTION_ASSIST`-trigger gebruikt), wordt de app geopend; als je "Hey Google, ask OpenClaw `<prompt>`" zegt, komt dit overeen met het gedeclareerde App Actions-querypatroon van de app en wordt de prompt zonder deze automatisch te verzenden in het chatinvoerveld geplaatst.

Dit gebruikt Android **App Actions** (mogelijkheid `shortcuts.xml`) die in het appmanifest zijn gedeclareerd. Er is geen configuratie aan de Gateway-zijde nodig — de assistentintentie wordt volledig door de Android-app afgehandeld.

<Note>
De beschikbaarheid van App Actions hangt af van het apparaat, de versie van Google Play Services en of de gebruiker OpenClaw als standaardassistent-app heeft ingesteld.
</Note>

## Meldingen doorsturen

Android kan apparaatmeldingen als `node.event`-items doorsturen naar de Gateway. Dit wordt **op het apparaat** geconfigureerd, in het instellingenvenster van de app — niet in de configuratie van Gateway/`openclaw.json`.

| Instelling                     | Beschrijving                                                                                                                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Meldingsgebeurtenissen doorsturen | Hoofdschakelaar. Standaard uitgeschakeld; vereist dat eerst toegang tot de meldingslistener wordt verleend.                                                                                           |
| Pakketfilter                   | **Toestaanlijst** (alleen vermelde pakket-ID's worden doorgestuurd) of **Blokkeerlijst** (standaard: alle pakketten behalve vermelde ID's). Het eigen pakket van OpenClaw wordt in de blokkeerlijstmodus altijd uitgesloten om doorstuurlussen te voorkomen. |
| Stille uren                    | Lokaal begin-/eindtijdvenster in HH:mm-indeling waarin doorsturen wordt onderdrukt. Standaard uitgeschakeld; na inschakeling zijn de standaardwaarden `22:00`-`07:00`. |
| Max. gebeurtenissen/minuut     | Snelheidslimiet per apparaat voor doorgestuurde meldingen. Standaard 20.                                                                                                                                |
| Sessiesleutel voor routering   | Optioneel. Zet doorgestuurde meldingsgebeurtenissen vast in een specifieke sessie in plaats van de standaardmeldingsroute van het apparaat.                                                            |

<Note>
Voor het doorsturen van meldingen is de Android-machtiging voor de meldingslistener vereist. De app vraagt hier tijdens de configuratie om.
</Note>

Meldingen van WhatsApp, WhatsApp Business, Telegram, Telegram X, Discord en Signal worden altijd uitgesloten. Hun berichten worden al beheerd door systeemeigen OpenClaw-kanaalsessies; als de Android-melding als afzonderlijke Node-gebeurtenis wordt doorgestuurd, kan een antwoord naar het verkeerde gesprek worden gerouteerd.

## Gerelateerd

- [iOS-app](/nl/platforms/ios)
- [Nodes](/nl/nodes)
- [Problemen met Android-nodes oplossen](/nl/nodes/troubleshooting)
