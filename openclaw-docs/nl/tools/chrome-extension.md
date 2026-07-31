---
read_when:
    - Je wilt dat een agent je echte, ingelogde Chrome vanaf je telefoon bedient
    - Je krijgt steeds de Chrome-melding 'Allow remote debugging?' terwijl er niemand achter de computer zit
    - Je wilt het beveiligingsmodel begrijpen van het overnemen van de browser via de extensie
summary: 'Chrome-extensie: laat OpenClaw je ingelogde Chrome aansturen zonder prompt voor foutopsporing op afstand'
title: Chrome-extensie
x-i18n:
    generated_at: "2026-07-27T06:14:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Chrome-extensie

Met de OpenClaw Chrome-extensie kan een agent je **aangemelde Chrome-tabbladen**
besturen zonder een afzonderlijke beheerde browser te starten en **zonder**
Chrome's blokkerende melding "Allow remote debugging?".

Dit is van belang wanneer je OpenClaw vanaf een telefoon bedient (Telegram,
WhatsApp enzovoort): het [`user`-profiel](/nl/tools/browser#profiles-openclaw-user-chrome) maakt verbinding via
Chrome's poort voor foutopsporing op afstand, waardoor een toestemmingsvenster
op het bureaublad verschijnt waarop niemand kan klikken wanneer je niet aanwezig
bent. De extensie gebruikt in plaats daarvan de `chrome.debugger`-API, zodat de
enige aanwijzing op de pagina Chrome's wegklikbare banner "OpenClaw started
debugging this browser" is.

Dit is dezelfde opzet die wordt gebruikt door de Chrome-extensies van
Anthropic's Claude en OpenAI's Codex.

## Hoe het werkt

Drie onderdelen:

- **Browserbesturingsservice** (Gateway- of nodehost): de API die de
  `browser`-tool aanroept.
- **Extensierelay** (loopback-WebSocket): een kleine server die door de
  besturingsservice op `127.0.0.1` wordt gestart. Deze biedt OpenClaw een
  Chrome DevTools Protocol-eindpunt en communiceert met de extensie. Beide zijden
  verifiëren zich met een hostlokaal token (zie hieronder).
- **OpenClaw Chrome-extensie** (MV3): maakt met `chrome.debugger` verbinding
  met tabbladen, stuurt CDP-verkeer door en beheert de **OpenClaw-tabbladgroep**.

OpenClaw ziet en bestuurt alleen tabbladen die zich in de
**OpenClaw-tabbladgroep** bevinden. De groep vormt de toestemmingsgrens: sleep
een tabblad erin om het te delen en sleep het eruit (of klik op de werkbalkknop)
om de toegang onmiddellijk in te trekken.

## Installeren en koppelen

1. Geef het pad naar de uitgepakte extensie weer:

   ```bash
   openclaw browser extension path
   ```

2. Open `chrome://extensions`, schakel **Developer mode** in, klik op **Load
   unpacked** en selecteer de weergegeven map.

3. Geef de koppelingsreeks weer:

   ```bash
   openclaw browser extension pair
   ```

4. Klik op het OpenClaw-pictogram in de werkbalk en plak de koppelingsreeks in
   de pop-up. De badge verandert in **AAN** wanneer de extensie verbinding maakt
   met de relay.

Het koppelingstoken is een **hostlokaal geheim** dat bij het eerste gebruik wordt
aangemaakt en onder `credentials/` in de statusmap wordt opgeslagen (modus
`0600`). Elke machine waarop een browser wordt uitgevoerd — de
Gateway-host en elke browsernodehost — heeft een eigen token, zodat er geen
inloggegevens tussen machines hoeven te worden uitgewisseld. Verwijder het
bestand `browser-extension-relay.secret` en koppel opnieuw om het token te roteren.

## Gebruiken

Selecteer het ingebouwde `chrome`-profiel in een aanroep van de
`browser`-tool of stel het in als standaardprofiel:

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- Deel een tabblad: klik op de OpenClaw-werkbalkknop van dat tabblad (het
  wordt aan de OpenClaw-tabbladgroep toegevoegd) of sleep een willekeurig
  tabblad naar de groep.
- De agent kan ook nieuwe tabbladen openen; deze worden automatisch aan de
  groep toegevoegd.
- Trek de toegang in: klik nogmaals op de knop, sleep het tabblad uit de groep
  of sluit Chrome's foutopsporingsbanner. De agent verliest onmiddellijk de
  toegang tot dat tabblad.

### Zijpaneel voor tabbladcopilot

Klik na het koppelen van de extensie in de werkbalkpop-up op
**Tabbladcopilot openen**. OpenClaw configureert `sidepanel.html` voor precies
dat Chrome-tabblad; het manifest heeft geen globaal zijpaneelpad. Elk tabblad
krijgt daarom een afzonderlijk paneeldocument, een afzonderlijke
Gateway-sessie, een afzonderlijk berichtabonnement en een getypeerde koppeling
met de browsertool.

Het paneel neemt de pagina-URL, titel, DOM of zichtbare tekst niet op in je
bericht. Het verzendt alleen de tekst die je typt. Browseracties bevatten een
afzonderlijke, door de Gateway geverifieerde koppeling met het Chrome-tabblad en
het CDP-doel, en de browsertool weigert pogingen om dat doel te vervangen of
browserbrede acties te gebruiken. Antwoorden blijven in het paneel
(`deliver: false`); ze nemen geen route van Telegram, Discord of een ander
kanaal over.

De copilot is een speciaal gekoppeld Gateway-apparaat met de bereiken
`operator.read` en `operator.write`. Controleer en keur de aanvraag bij het
eerste gebruik goed:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

De extensie behoudt die apparaatidentiteit en het door de Gateway uitgegeven
apparaattoken, beperkt tot het canonieke Gateway-eindpunt dat ze heeft
uitgegeven. Door een andere Gateway te koppelen, worden afzonderlijk beheer van
identiteit, token en sessie aangemaakt; inloggegevens en sessies worden nooit
tussen eindpunten hergebruikt. De extensie slaat het gedeelde Gateway-geheim
niet permanent op. Een paneel kan zich alleen abonneren op de eigen
tabbladsessies en de Gateway filtert die gebeurtenissen vóór aflevering.

Als de Gateway-verbinding tijdens een uitvoering wegvalt, behoudt de extensie
duurzaam het beheer over die uitvoerings-ID. Na het opnieuw verbinden breekt de
extensie de onopgeloste uitvoering af voordat een paneel opnieuw wordt
ingeschakeld en laadt vervolgens de transcriptgeschiedenis opnieuw. Deze
fail-closed-stap voorkomt dat browseracties tijdens een onderbreking in de
aflevering onzichtbaar doorgaan.

Wanneer een tabblad wordt gesloten, wordt het actieve abonnement onmiddellijk
verwijderd, wordt elke zichtbare uitvoering afgebroken en wordt de sessie van
dat tabblad als gearchiveerd gemarkeerd. Als de Gateway tijdelijk offline is,
slaat de extensie de wachtende archivering op en probeert ze het alleen opnieuw
wanneer hetzelfde Gateway-eindpunt opnieuw verbinding maakt; ze verzendt nooit
een archiveringsverzoek naar een andere Gateway. Na een browsercrash archiveert
de volgende start de sessies die door de vorige browserinstantie zijn
achtergelaten. Gearchiveerde sessies weigeren nieuw werk, terwijl hun
transcripten beschikbaar blijven in de sessiegeschiedenis.
Browsercopilot-sleutels zijn threadsessies, zodat ze door normaal onderhoud op
basis van ouderdom en aantal vermeldingen behouden blijven. Het schijfquotum
per agent voor sessies blijft van toepassing (standaard `2gb`) en
kan onder druk de oudste sessies verwijderen; zie
[sessieonderhoud](/nl/reference/session-management-compaction#store-maintenance-and-disk-controls).

Het zijpaneel vereist momenteel een door de Gateway gehoste extensierelay of een
rechtstreekse externe Gateway-relay. Een loopback-relay op een browsernode kan
nog niet de noderoute bieden die voor de getypeerde tabbladkoppeling nodig is,
dus het paneel weigert die topologie in plaats van terug te vallen op
browserbrede routering.

## Een pagina naar OpenClaw verzenden

Gebruik **Pagina naar OpenClaw verzenden** in de werkbalkpop-up om leesbare
paginatekst met je hoofd-OpenClaw-sessie te delen. Je kunt een optionele notitie
toevoegen, het contextmenu van de pagina of selectie gebruiken of op
`Alt+Shift+S` drukken. OpenClaw geeft de voorkeur aan je huidige selectie
wanneer die bestaat, zet het gedeelde materiaal als systeemgebeurtenis in de
wachtrij en activeert de hoofdsessie onmiddellijk.

Het tabblad hoeft zich niet in de OpenClaw-tabbladgroep te bevinden. Dit is
eenmalig, expliciet delen: niets anders op de pagina wordt blootgesteld en het
verleent geen doorlopende toegang. Google Docs-documenten worden met je
aangemelde browsersessie als platte tekst geëxporteerd, zonder configuratie van
de Google API. Threads van X en Twitter worden zonder de omliggende
interface-elementen geëxtraheerd.

Paginatekst wordt binnen OpenClaw's veiligheidsgrens voor externe inhoud
geplaatst. Je optionele notitie blijft als je eigen instructie buiten die
grens. Paginatekst en selecties zijn beperkt tot ongeveer 120,000 tekens en
bevatten een afkappingsmarkering wanneer ze zijn ingekort.

Pagina's delen werkt wanneer de extensierelay door de Gateway wordt gehost, met
koppeling op dezelfde host of rechtstreekse koppeling met de
`wss://`-Gateway. Door nodes gehoste relays geven voorlopig een
duidelijke foutmelding. Open `chrome://extensions/shortcuts` om de sneltoets opnieuw toe te
wijzen.

## Extern / tussen machines

Chrome hoeft niet op de Gateway-host te worden uitgevoerd. Drie topologieën
werken:

- **Dezelfde host** (Gateway + Chrome op één machine): koppel op die machine
  met `openclaw browser extension pair`. De relay is uitsluitend via loopback bereikbaar.
  Als de lokale Gateway TLS gebruikt, geef je de hostnaam van het certificaat
  expliciet door met `--gateway-url wss://gateway-host.example`; bij het koppelen wordt nooit een
  loopback-IP-adres gebruikt als vervanging.
- **Rechtstreeks naar een externe Gateway** (Chrome op je laptop, Gateway op
  een VPS en **niets anders op de laptop**): voer op de Gateway
  `openclaw browser extension pair --gateway-url wss://your-gateway.example.com` uit.
  Hiermee wordt een `wss://…/browser/extension#<secret>`-reeks weergegeven; laad en koppel de
  extensie op de laptop. De extensie maakt **rechtstreeks verbinding met de
  Gateway** via `wss://` — zonder installatie van OpenClaw, Node, CLI
  of een geopende inkomende poort op de laptop. Dit is het pad voor beheerde
  hosting.
- **Via een browsernodehost** (Chrome op een machine waarop al een
  OpenClaw-node wordt uitgevoerd): voer `pair` op de node uit en
  koppel lokaal; de Gateway stuurt browseracties via de bestaande
  geverifieerde nodeverbinding door naar de node.

Het koppelingsgeheim is per host (in het rechtstreekse geval dat van de Gateway)
en wordt door de `/browser/extension`-route van de Gateway gevalideerd. Bied de
Gateway voor het rechtstreekse pad via TLS (`wss://`) aan, zodat het
koppelingsgeheim en CDP-verkeer worden versleuteld. Het geheim blijft in het
URL-fragment van de koppelingsreeks en wordt tijdens de WebSocket-handshake als
subprotocolreferentie aangeboden, zodat normale toegangslogboeken van proxy's
het niet in de aanvraag-URL ontvangen. Zorg ervoor dat een eventuele reverse
proxy de standaardheader `Sec-WebSocket-Protocol` behoudt.

## Diagnostiek

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

`doctor` meldt dat de controle van de **Chrome-extensierelay** mislukt
totdat in de extensiepop-up **Verbonden** wordt weergegeven.

## Beveiligingsmodel

- De relay bindt uitsluitend aan loopback; beide WebSocket-zijden worden met
  het afgeleide token geverifieerd en de oorsprong aan de extensiezijde wordt
  gecontroleerd op `chrome-extension://`.
- Bij rechtstreekse Gateway-koppeling wordt het relaytoken niet in de
  aanvraag-URL geaccepteerd; de meegeleverde extensie neemt het in plaats
  daarvan op in de lijst met WebSocket-subprotocollen.
- De agent kan alleen tabbladen in de **OpenClaw-tabbladgroep** zien en
  besturen. Je andere tabbladen blijven privé.
- Uitvoeringen in het zijpaneel zijn dubbel beperkt: voor Gateway-aflevering
  wordt een toelatingslijst per sessie gebruikt en browsertools dwingen de
  koppeling met het Chrome-tabblad/-doel af die buiten de prompt wordt
  meegestuurd.
- Vergeleken met het `user`-profiel (Chrome MCP), dat je volledige
  aangemelde browser blootstelt zodra je de melding voor foutopsporing op
  afstand goedkeurt, beperkt de extensie het gedeelde oppervlak tot een
  tabbladgroep die je in één oogopslag kunt beheren.

Zie ook: [Browser](/nl/tools/browser) voor het volledige profielmodel en de
beheerde profielen `openclaw` en Chrome MCP `user`.
