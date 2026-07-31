---
read_when:
    - Je wilt de Gateway vanuit een browser bedienen
    - Je wilt toegang tot het Tailnet zonder SSH-tunnels
sidebarTitle: Control UI
summary: Browsergebaseerde beheerinterface voor de Gateway (chat, activiteit, nodes, configuratie)
title: Bedieningsinterface
x-i18n:
    generated_at: "2026-07-27T06:17:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 069bad7f3c8fce46759893e16d2dac86047c0929d6d866d25ce3b080204c1180
    source_path: web/control-ui.md
    workflow: 16
---

De Control UI is een kleine **Vite + Lit**-app met één pagina die door de Gateway wordt aangeboden:

- standaard: `http://<host>:18789/`
- optioneel voorvoegsel: stel `gateway.controlUi.basePath` in (bijv. `/openclaw`)

De app communiceert **rechtstreeks met de Gateway WebSocket** op dezelfde poort.

Terwijl je een actieve sessie bekijkt, kan de Gateway het hulpprogrammamodel van die agent gebruiken om een beknopt statusoverzicht te produceren. Chat toont dit als een statuslabel van één regel dat uitvouwt tot een kaart met de beoordeling, de voortgang van het plan, pull requests en de verstreken tijd. De kaart kan eenmaal uitvouwen wanneer een uitvoering vastloopt of invoer nodig heeft; de zijchat `/btw` heeft voorrang op de uitgevouwen kaart.

De uitgevouwen kaart accepteert ook korte vragen over de uitvoering. Antwoorden gebruiken alleen het huidige overzicht en de opgeschoonde, begrensde notities van de waarnemer, blijven voor die sessie in de browser en komen nooit in de uitvoering van de hoofdagent terecht of onderbreken deze. Als de waarnemingen het antwoord niet bevatten, zegt de waarnemer dat die het niet kan weten.

Nadat het eerste overzicht is binnengekomen, bepaalt dit de subtitel van die uitvoering in de zijbalk in plaats van heuristische liveactiviteit. Een definitief overzicht met voltooid of mislukt blijft zichtbaar zolang de sessie ongelezen is; daarna krijgt de rij weer de normale werksubtitel.

Sessiewaarneming is standaard ingeschakeld. In **Settings > Appearance > Sidebar** kun je deze voor de hele Gateway uitschakelen, het vastgestelde kleine model en de herkomst ervan bekijken, automatische routering kiezen, hulptaken uitschakelen of expliciet een `agents.defaults.utilityModel` selecteren. De overeenkomstige configuratieopties zijn `gateway.controlUi.sessionObserver: false` en `agents.defaults.utilityModel: ""`.

## Snel openen (lokaal)

Als de Gateway op dezelfde computer draait, open je [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (of [http://localhost:18789/](http://localhost:18789/)).

Als de pagina niet wordt geladen, start je eerst de Gateway: `openclaw gateway`.

<Note>
Bij native Windows-LAN-bindingen kan Windows Firewall of door de organisatie beheerd groepsbeleid de geadverteerde LAN-URL nog steeds blokkeren, zelfs wanneer `127.0.0.1` op de Gateway-host werkt. Voer `openclaw gateway status --deep` uit op de Windows-host; dit rapporteert waarschijnlijk geblokkeerde poorten, niet-overeenkomende profielen en lokale firewallregels die mogelijk door beleid worden genegeerd.
</Note>

Authenticatie wordt tijdens de WebSocket-handshake geleverd via:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale Serve-identiteitsheaders wanneer `gateway.auth.allowTailscale: true`
- identiteitsheaders van vertrouwde proxy's wanneer `gateway.auth.mode: "trusted-proxy"`

Gateway-authenticatie vindt plaats vóór het koppelen van apparaten. Een rechtstreekse loopbackverbinding omzeilt token- of wachtwoordauthenticatie niet. Het instellingenpaneel van het dashboard bewaart een token voor de huidige browsertabsessie en de geselecteerde Gateway-URL; wachtwoorden worden niet bewaard. Na het koppelen kan de browser bij latere verbindingen het opgeslagen token per apparaat gebruiken.

Tijdens de onboarding wordt doorgaans een Gateway-token geconfigureerd voor authenticatie met een gedeeld geheim. Als de Gateway in tokenmodus start zonder een geconfigureerd token, genereert deze in plaats daarvan een tijdelijk runtimetoken voor dat proces. Het runtimetoken wordt niet naar de configuratie geschreven, dus `openclaw config get gateway.auth.token` kan het niet ophalen en een loopbackbrowser zonder dat token wordt geweigerd. Voer `openclaw doctor --generate-gateway-token` uit, start de Gateway opnieuw en plak vervolgens het geconfigureerde token in de instellingen van de Control UI. Wachtwoordauthenticatie werkt als alternatief wanneer `gateway.auth.mode` `"password"` is.

## Apparaten koppelen (eerste verbinding)

Nadat Gateway-authenticatie is geslaagd, vereist verbinding maken vanuit een nieuwe browser of vanaf een nieuw apparaat doorgaans een **eenmalige koppelingsgoedkeuring**, weergegeven als `disconnected (1008): pairing required`.

<Warning>
Bij een rechtstreekse upgrade vanaf een release die de uitgefaseerde
noodinstelling `gateway.controlUi.dangerouslyDisableDeviceAuth=true` gebruikte,
houdt OpenClaw toegang tot de Control UI met token-/wachtwoordauthenticatie of
authenticatie via een vertrouwde proxy beschikbaar, uitsluitend om het koppelen
te herstellen. Als de browser gewone HTTP gebruikt en geen apparaatidentiteit kan maken,
open je deze eerst opnieuw via HTTPS of localhost. Klik vervolgens op **Secure this browser** in
de waarschuwingsbanner. De Gateway hervat de normale handhaving van apparaatauthenticatie pas
nadat een ondertekende browser expliciet is gekoppeld; deze maakt of keurt nooit een
identiteit goed voor een browser zonder apparaatidentiteit. De overgang is niet beschikbaar wanneer
al een ander beheerdersapparaat is gekoppeld. Zowel het opstarten van de Gateway als
`openclaw doctor --fix` meldt deze migratie expliciet in plaats van
de oude sleutel stilzwijgend te verwijderen.
</Warning>

<Steps>
  <Step title="Openstaande aanvragen weergeven">
    ```bash
    openclaw devices list
    ```
  </Step>
  <Step title="Goedkeuren op aanvraag-ID">
    ```bash
    openclaw devices approve <requestId>
    ```
  </Step>
</Steps>

Als de browser het koppelen opnieuw probeert met gewijzigde authenticatiegegevens (rol/bereiken/openbare sleutel), wordt de vorige openstaande aanvraag vervangen en wordt een nieuwe `requestId` gemaakt; voer `openclaw devices list` opnieuw uit voordat je deze goedkeurt.

Het overschakelen van een reeds gekoppelde externe browser van leestoegang naar schrijf-/beheerderstoegang wordt behandeld als een upgrade van de goedkeuring, niet als een stilzwijgende herverbinding: OpenClaw houdt de oude goedkeuring actief, blokkeert de herverbinding met ruimere toegang en vraagt je om de nieuwe reeks bereiken expliciet goed te keuren. Een geschikte rechtstreekse loopbackverbinding van de Control UI kan de upgrade stilzwijgend goedkeuren nadat deze is geauthenticeerd.

Na goedkeuring wordt het apparaat onthouden en is hernieuwde goedkeuring niet nodig, tenzij je deze intrekt met `openclaw devices revoke --device <id> --role <role>`. Zie [CLI voor apparaten](/nl/cli/devices) voor tokenrotatie, intrekking en de goedkeuringsflow bij de eerste uitvoering van Paperclip / `openclaw_gateway`.

<Note>
- Rechtstreekse lokale Control UI-verbindingen vanaf een loopback-TCP-peer (`127.0.0.1` of `::1`, doorgaans bereikbaar als `localhost`) zonder doorgestuurde/proxyheaders kunnen het koppelen van apparaten alleen automatisch goedkeuren nadat Gateway-authenticatie is geslaagd en de browser een apparaatidentiteit aanbiedt. In token-/wachtwoordmodus heeft de eerste verbinding nog steeds het geconfigureerde gedeelde geheim nodig; deze automatische goedkeuring omzeilt het token niet.
- Rechtstreekse loopback heeft alleen geen gedeeld geheim nodig wanneer `gateway.auth.mode: "none"` expliciet is geconfigureerd. Hierdoor wordt Gateway-authenticatie uitgeschakeld en dit is niet de aanbevolen configuratie voor de Control UI. Tailscale Serve en modi met een vertrouwde proxy kunnen een geplakt gedeeld geheim alleen vermijden wanneer hun respectieve identiteitscontroles slagen.
- Tailscale Serve kan de koppelingsronde voor Control UI-beheerderssessies overslaan wanneer `gateway.auth.allowTailscale: true`, de Tailscale-identiteit wordt geverifieerd en de browser zijn apparaatidentiteit aanbiedt. Browsers zonder apparaatidentiteit en verbindingen met een Node-rol volgen nog steeds de normale apparaatcontroles.
- Rechtstreekse Tailnet-bindingen en LAN-browserverbindingen vereisen nog steeds expliciete goedkeuring. Browserprofielen zonder apparaatidentiteit kunnen automatische loopbackgoedkeuring niet gebruiken.
- Elk browserprofiel genereert een unieke apparaat-ID, dus als je van browser wisselt of browsergegevens wist, moet je opnieuw koppelen.

</Note>

## Een mobiel apparaat koppelen

Een reeds gekoppelde beheerder kan de QR-code voor de iOS-/Android-verbinding maken zonder een terminal te openen:

<Steps>
  <Step title="Mobiele koppeling openen">
    Selecteer **Devices** en klik vervolgens op **Pair mobile device** in de kaart **Devices**.
  </Step>
  <Step title="De telefoon verbinden">
    Open in de mobiele OpenClaw-app **Settings** → **Gateway** en scan de QR-code. Je kunt in plaats daarvan de installatiecode kopiëren en plakken.
  </Step>
  <Step title="De verbinding bevestigen">
    De officiële iOS-/Android-app maakt automatisch verbinding. Als **Pending approval** een aanvraag toont, controleer je de rol en bereiken voordat je deze goedkeurt.
  </Step>
</Steps>

Voor het maken van een installatiecode is `operator.admin` vereist; de knop is uitgeschakeld voor sessies zonder deze optie. Een installatiecode bevat een kortlevende bootstrapreferentie, dus behandel de QR-code en de gekopieerde code als een wachtwoord zolang ze geldig zijn. Voor koppeling op afstand moet de Gateway worden omgezet naar `wss://` (bijvoorbeeld via Tailscale Serve/Funnel); gewone `ws://` is beperkt tot loopback- en privé-LAN-adressen. Zie [Koppelen](/nl/channels/pairing#pair-from-the-control-ui-recommended) voor de volledige beveiligings- en fallbackdetails.

## Persoonlijke identiteit (browserlokaal)

De Control UI ondersteunt een persoonlijke identiteit per browser (weergavenaam en avatar) die aan uitgaande berichten wordt gekoppeld voor toeschrijving in gedeelde sessies. Deze wordt opgeslagen in de browser, is beperkt tot het huidige browserprofiel en wordt niet met andere apparaten gesynchroniseerd of server-side bewaard, afgezien van de normale auteursmetadata in het transcript van berichten die je verzendt. Als je sitegegevens wist of van browser wisselt, wordt deze weer leeg.

De overschrijving van de assistentavatar volgt hetzelfde browserlokale patroon: geüploade overschrijvingen worden lokaal over de door de Gateway vastgestelde identiteit gelegd en worden nooit via `config.patch` heen en weer gestuurd. Het gedeelde configuratieveld `ui.assistant.avatar` blijft beschikbaar voor clients zonder UI die rechtstreeks naar het veld schrijven.

## Endpoint voor runtimeconfiguratie

De Control UI haalt de runtime-instellingen op via `/control-ui-config.json`, relatief ten opzichte van het basispad van de Control UI van de Gateway (bijvoorbeeld `/__openclaw__/control-ui-config.json` onder basispad `/__openclaw__/`). Dit endpoint wordt beveiligd door dezelfde Gateway-authenticatie als de rest van het HTTP-oppervlak: niet-geauthenticeerde browsers kunnen het niet ophalen en voor een geslaagde ophaalactie is een geldig Gateway-token/-wachtwoord, een Tailscale Serve-identiteit of een identiteit van een vertrouwde proxy vereist.

## Status van de Gateway-host

Open **Settings → General** om de kaart **Gateway Host** te zien met de Gateway-machine, het LAN-adres, het besturingssysteem, de runtime, de actieve tijd, de CPU-belasting, het geheugen en de schijfruimte van het statusvolume. De kaart wordt elke 10 seconden vernieuwd zolang deze zichtbaar is, via de Gateway-RPC `system.info`, waarvoor het bereik `operator.read` vereist is. Oudere Gateways en verbindingen zonder dat bereik tonen de kaart niet.

## Taalondersteuning

De Control UI wordt bij de eerste keer laden gelokaliseerd op basis van de landinstelling van je browser. Als je dit later wilt overschrijven, open je **Settings -> General -> Language** (de keuzelijst staat op de pagina General, niet onder Appearance).

- Ondersteunde landinstellingen: `en`, `ar`, `de`, `es`, `fa`, `fr`, `hi`, `id`, `it`, `ja-JP`, `ko`, `nl`, `pl`, `pt-BR`, `ru`, `th`, `tr`, `uk`, `vi`, `zh-CN`, `zh-TW`
- Niet-Engelse vertalingen worden lui geladen in de browser.
- De geselecteerde landinstelling wordt opgeslagen in de browser en bij toekomstige bezoeken opnieuw gebruikt.
- Ontbrekende vertaalsleutels vallen terug op Engels.

Documentatievertalingen worden voor dezelfde reeks niet-Engelse landinstellingen gegenereerd, maar de ingebouwde taalkiezer van Mintlify op de documentatiesite vermeldt alleen landinstellingscodes die Mintlify accepteert. Thaise (`th`) en Perzische (`fa`) documentatie wordt nog steeds in de publicatierepository gegenereerd; deze verschijnt mogelijk pas in die keuzelijst wanneer Mintlify deze codes ondersteunt.

## Weergavethema's

Het paneel Appearance bevat de ingebouwde thema's Claw, Knot en Dash (Claw is de standaard), plus één browserlokaal importslot voor tweakcn. Om een thema te importeren, open je de [tweakcn-editor](https://tweakcn.com/editor/theme), kies of maak je een thema, klik je op **Share** en plak je de gekopieerde link in Appearance. De importfunctie accepteert ook register-URL's voor `https://tweakcn.com/r/themes/<id>`, editor-URL's zoals `https://tweakcn.com/editor/theme?theme=amethyst-haze`, relatieve paden voor `/themes/<id>`, onbewerkte thema-ID's en namen van standaardthema's zoals `amethyst-haze`.

Geïmporteerde thema's worden alleen in het huidige browserprofiel opgeslagen; ze worden niet naar de Gateway-configuratie geschreven en niet tussen apparaten gesynchroniseerd. Als je het geïmporteerde thema vervangt, wordt dat ene lokale slot bijgewerkt; als je het wist terwijl het actief was, wordt teruggeschakeld naar Claw.

Appearance bevat ook de instelling Text size. Deze is van toepassing op chattekst, tekst in het invoerveld, toolkaarten en chatzijbalken, en houdt tekstinvoervelden op minimaal 16px zodat Safari op mobiele apparaten niet automatisch inzoomt wanneer ze de focus krijgen.

Voorkeuren voor thema, themamodus, tekstgrootte, taal en chatweergave worden gesynchroniseerd via de Gateway-configuratie (`ui.prefs`), zodat ze je op al je apparaten volgen en agents ze via de goedkeuringspoort kunnen wijzigen — verbonden clients passen wijzigingen direct toe via de `config.changed`-melding van de Gateway. Elke browser bewaart een lokale kopie om direct te kunnen opstarten; clients die de configuratie niet kunnen schrijven (viewerscope, offline), houden wijzigingen lokaal op het apparaat. Zie [Configuratiereferentie](/nl/gateway/configuration-reference#ui).

## OpenClaw-systeemonderhoud

Open **Instellingen → Vraag OpenClaw** om met de agent voor systeemconfiguratie en -herstel te praten. Buiten de onboarding kan deze pagina per bezoek maximaal één wegklikbare gebeurtenischip tonen. De pagina blijft stil bij regulier Gateway-verkeer en reageert alleen op statusmomentopnamen die melden dat het opnieuw laden van de configuratie is uitgeschakeld, dat een geconfigureerd kanaal is verbroken of verslechterd, dat een kanaalcontrole is mislukt of dat kanaalreferenties niet beschikbaar zijn. Een nieuwere gebeurtenis vervangt de wachtende chip alleen als die ernstiger is; als je de chip sluit of gebruikt, worden gebeurtenisprompts voor dat bezoek onderdrukt. Als je op de chip klikt, wordt de diagnosevraag als een echt `openclaw.chat`-bericht verzonden, zodat het transcript het verzoek vastlegt en OpenClaw de diagnose uitvoert. Tijdens de onboarding worden deze gebeurtenischips nooit getoond.

## Plugins beheren

Open **Plugins** in de zijbalk of gebruik `/settings/plugins` ten opzichte van het
geconfigureerde basispad van de Control UI om plugins te bekijken en beheren zonder
de Control UI te verlaten. Een basispad van `/openclaw` gebruikt
bijvoorbeeld `/openclaw/settings/plugins`. De pagina is altijd beschikbaar, zelfs wanneer alle
optionele plugins zijn uitgeschakeld.

Plugins is een centrale pagina met vier tabbladen: **Geïnstalleerd** en **Ontdekken** beheren
plugincode op `/settings/plugins`, **Skills** bevat het Skills-beheer per agent op
`/skills` en **Workshop** bevat de beoordeling van Skill Workshop-voorstellen op
`/skills/workshop`. Elk tabblad behoudt zijn eigen URL en de zijbalk toont
één vermelding Plugins voor alle tabbladen.

Het tabblad **Geïnstalleerd** toont de volledige lokale inventaris, gegroepeerd op categorie, met
overzichtsaantallen. Elke rij opent een detailweergave; via het overloopmenu (`…`)
kan de plugin worden in- of uitgeschakeld en is **Verwijderen** beschikbaar voor extern geïnstalleerde plugins.
Het vermeldt ook geconfigureerde [MCP-servers](/nl/cli/mcp) en ondersteunt het rechtstreeks toevoegen, uitschakelen
en verwijderen ervan. Dezelfde serverbediening staat onder **Instellingen → MCP**.
Het tabblad **Ontdekken** is de winkel: uitgelichte plugins die bij OpenClaw zijn inbegrepen,
officiële externe plugins en MCP-connectors voor populaire diensten die met één klik kunnen worden toegevoegd.
Als je in het zoekvak typt, wordt
[ClawHub](https://clawhub.ai/plugins) rechtstreeks doorzocht en wordt een sectie **Van ClawHub**
toegevoegd met aantallen downloads en badges voor bronverificatie. Deep links kunnen
met `/settings/plugins?tab=discover` rechtstreeks naar de winkel verwijzen.

Het tabblad **Skills** bevat het Skills-statusrapport, schakelaars voor in- en uitschakelen, invoer van
API-sleutels en rechtstreeks zoeken naar Skills in ClawHub, beperkt tot de geselecteerde agent. Het
tabblad **Workshop** bevat het Skill Workshop-bord en de beoordelingsflow van Vandaag voor
[Skill-voorstellen](/nl/tools/skill-workshop). **Ideeën voor Skills zoeken** beoordeelt een begrensd
venster met inhoudelijke sessies, van nieuw naar oud, en laat eventuele resultaten achter als
wachtende voorstellen. Het paneel toont de cumulatieve dekking; **Eerder werk scannen**
gaat verder vanaf de opgeslagen cursor en verandert in **Nieuw werk scannen** nadat de oudere
geschiedenis is uitgeput. Handmatige beoordeling van de geschiedenis werkt terwijl autonoom zelfleren
is uitgeschakeld en gebruikt het geconfigureerde model van de geselecteerde agent.

Inbegrepen plugins zijn al aanwezig op de Gateway en tonen **Inschakelen** of
**Uitschakelen** in plaats van **Installeren**. Workboard is bijvoorbeeld inbegrepen bij
OpenClaw, maar standaard uitgeschakeld, waardoor de actie **Inschakelen** is. Gebundelde plugins
kunnen niet worden verwijderd, alleen uitgeschakeld.

Voor het lezen van de catalogus en doorzoeken van ClawHub is `operator.read` vereist. Voor het installeren,
inschakelen, uitschakelen of verwijderen van een plugin en het wijzigen van MCP-servers is
`operator.admin` vereist; deze acties blijven uitgeschakeld voor operators met alleen-lezenrechten.

Installaties vanuit ClawHub lopen via de Gateway en hanteren dezelfde controles voor vertrouwen, integriteit
en plugininstallatiebeleid als andere installaties die via de Gateway verlopen. Voor het installeren
of verwijderen van plugincode moet de Gateway opnieuw worden gestart. Het in- of uitschakelen van een
geïnstalleerde plugin kan zonder herstart worden toegepast wanneer de plugin en de huidige
Gateway-runtime dit ondersteunen; anders meldt de UI dat een herstart
vereist is. Voor MCP-connectors met OAuth-ondersteuning is na toevoeging eenmalig
`openclaw mcp login <name>` vanuit de CLI nodig.

De pagina richt zich bewust op inventaris, ontdekking, installatie, inschakeling
en verwijdering. Gebruik [`openclaw plugins`](/nl/cli/plugins) voor willekeurige npm-, git- of
lokale-padbronnen, updates en geavanceerde pluginconfiguratie.

## Apps en extensies

Open **Apps** via het menu **Meer** in de zijbalk, het opdrachtenpalet of het
agentmenu in de zijbalk (**Apps downloaden**), of gebruik `/apps` ten opzichte van het
geconfigureerde basispad van de Control UI. De pagina verzamelt installatielinks voor elk
OpenClaw-begeleidend platform: de [iOS](/nl/platforms/ios)- en
[Android](/nl/platforms/android)-apps, de meegeleverde Apple Watch- en Wear OS-begeleiders,
de desktopapps voor [macOS](/nl/platforms/macos), [Windows](/nl/platforms/windows)
en [Linux](/nl/platforms/linux), de
[Chrome-extensie](/nl/tools/chrome-extension), de ingebouwde Plugins-hub met
[ClawHub](https://clawhub.ai), en de Discord-community en documentatie.

## Navigatie in de zijbalk

De zijbalk organiseert alles rond de agent. De identiteitsrij bovenaan is de actieve agent; daaronder begint de sectie **Pagina's** met **Start** — de doorlopende hoofdsessie van de agent, voorzien van een badge met de ongelezen of actieve status — gevolgd door de vastgemaakte bestemmingen (standaard **Automatiseringen** en **Plugins**). Het bedieningselement voor aanpassing in de kop van Pagina's opent een menu met alle andere bestemmingen, waaronder **Gebruik** en door plugins geleverde tabbladen, plus **Vastgemaakte items bewerken**; als je met de rechtermuisknop op het navigatiegebied klikt, wordt de editor voor vastmaken rechtstreeks geopend. De onderstaande sessielijst is verdeeld in zones: **Threads** voor de chatsessies van de agent (de hoofdsessie blijft achter Start; door deze sessie gestarte sessies verschijnen hier als threads op het hoogste niveau en benoemde threads worden zonder typevoorvoegsel weergegeven), **Groepen** voor groeps- en kamergesprekken en **Coderen** voor sessies die zijn gekoppeld aan een beheerde worktree of uitvoeringsnode (rijen tonen een `repo ⎇ branch`-regel plus de nodehost), door ACP ondersteunde harness-sessies en de CLI-catalogi van Codex/Claude. Coderen is bij de eerste uitvoering ingeklapt en onthoudt je keuze; de ingeklapte kop behoudt het werkelijke aantal en toont een activiteitsindicator terwijl de sessies erin actief zijn. Aangepaste groepen (de sessie-`category`) en **Vastgemaakt**-rijen staan boven Threads, en als je een sessie aan een aangepaste groep toewijst, heeft dat altijd voorrang op de automatische zoneclassificatie. De kop van Threads bevat de sorteerbediening (Aangemaakt of Laatst bijgewerkt, Groeperen op en een opgeslagen **Status**-filter voor Actief, Gearchiveerd of Alles) en de **+** waarmee de pagina Nieuwe sessie wordt geopend. Gearchiveerde rijen blijven in de lijst staan, gedimd en voorzien van een archiefpictogram; ze dragen niet bij aan de ongelezen- of aandachtsstatus en blijven buiten de promotie op basis van afstamming. Als je een sessie opent, wordt de selectie gemarkeerd zonder de rijen opnieuw te ordenen. Bovenliggende sessies met recente onderliggende uitvoeringen tonen een uitvouwpictogram en het aantal onderliggende sessies; vouw dit uit om geneste onderliggende sessies, de actieve of beëindigde status en de runtime te bekijken zonder de zijbalk te verlaten. Als je een onderliggende sessie selecteert, wordt de chat ervan geopend en wordt het pad naar de bovenliggende sessies automatisch zichtbaar. Onderliggende rijen blijven buiten hoofdgroepering, vastmaken, slepen, meervoudige selectie en paginering; ingeklapte zones gebruiken geen deel van het zichtbare paginabudget. Sessies met nieuwe activiteit sinds ze voor het laatst zijn gelezen, tonen een ongelezen stip en worden als gelezen gemarkeerd wanneer je ze opent. Een agent kan ook een korte, vervallende statusregel publiceren en eventueel om aandacht vragen met een samengesteld amberkleurig pictogram; die melding wordt gewist wanneer je de sessie opent, het volgende bericht verzendt, de melding expliciet wist of de TTL verloopt. Levenscyclusstatussen van cloudworkers gebruiken een wereldbolbadge; lokale en teruggewonnen sessies hebben geen plaatsingsbadge, omdat lokale uitvoering de standaard is. Elke hoofdsessierij heeft een contextmenu (kebabknop of rechtermuisklik) met Vastmaken/Losmaken, Markeren als ongelezen/gelezen, Naam wijzigen, Forken, Naar groep verplaatsen (inclusief Nieuwe groep en Uit groep verwijderen), Archiveren of Uit archief halen en Verwijderen; in aanraaklay-outs blijven de directe bedieningselementen voor vastmaken en het menu zichtbaar. Met Cmd/Ctrl-klik schakel je hoofdrijen in of uit voor meervoudige selectie en met Shift-klik breid je de selectie uit over de zichtbare volgorde; als je vervolgens het menu van een geselecteerde rij opent, worden batchacties aangeboden (N markeren als ongelezen/gelezen, N naar groep verplaatsen, N archiveren, N verwijderen) die op elke geselecteerde sessie worden toegepast, met één bevestiging voor batchverwijdering. Sleep een hoofdsessie naar **Vastgemaakt** om deze vast te maken, of naar een aangepaste groep om deze te verplaatsen. Koppen van aangepaste groepen kunnen worden ingeklapt, uitgevouwen of versleept om de volgorde te wijzigen; groepsnamen en hun volgorde worden opgeslagen in de Gateway (`sessions.groups.*`), zodat ze je in alle browsers volgen, terwijl de ingeklapte status in het browserprofiel blijft. Groepskoppen hebben ook een menu (kebabknop of rechtermuisklik) met Groepsnaam wijzigen, Nieuwe groep en Groep verwijderen; als je een groep hernoemt of verwijdert, wordt elke bijbehorende sessie aan de serverzijde bijgewerkt, inclusief gearchiveerde sessies, en als je een groep verwijdert, blijven de sessies behouden en worden ze teruggeplaatst naar Threads.

## Pagina Nieuwe sessie

De **+** in de kop van de sessielijst in de zijbalk opent een concept op volledige pagina op `/new`: er wordt niets aangemaakt totdat je het eerste bericht verzendt. Met een uniforme **Plaats**-kiezer selecteer je de werkmap en, voor beheerders, de uitvoeringsbestemming: **Gateway · lokaal**, een gekoppelde node die `system.run` beschikbaar stelt, of een beschikbaar cloudprofiel. De map is standaard de agentworkspace; voor een ander absoluut Gateway-pad is `operator.admin` vereist, maar het kan rechtstreeks worden uitgevoerd zonder een Git-checkout te zijn. Wanneer de geselecteerde Gateway-map een Git-checkout is, biedt dezelfde kiezer optionele **Worktree**-isolatie met een basisbranchkiezer die wordt ondersteund door `worktrees.branches` (zonder fetch) en een optionele worktreenaam (de branch wordt `openclaw/<name>`). Cloudworkers vereisen dit beheerde worktreepad; gekoppelde nodes bieden het nooit aan. In de voettekst van de editor kies je het model en redeneerniveau van de nieuwe sessie. De schakelaar **Incognito** maakt een thread aan die alleen op het web bestaat en waarvan de sessievermelding, het transcript en de Compaction-status in het geheugen blijven totdat de Gateway opnieuw wordt gestart; OpenClaw slaat ook de automatische geheugensynchronisatie over. De agent behoudt zijn normale tools, zodat gegevens nog steeds kunnen worden opgeslagen via een expliciet opslagverzoek of door een tool aangestuurde schrijfactie naar een bestand. De modelprovider verwerkt de berichten nog steeds en auditmetadata zonder inhoud wordt nog steeds vastgelegd. Bij cloudstarts worden de model- en redeneerkeuzes opgeslagen voordat de sessie naar de worker wordt verzonden.

Op gateways met meerdere gebruikers kunnen alleen verbindingen met beheerdersscope incognitothreads maken of bekijken, en andere sessies kunnen ze niet bereiken via agentsessietools of het doorzoeken van transcripten. Incognito beschermt tegen opslag en andere gebruikers die via de Gateway werken, niet tegen de eigenaar of procesoperator van de Gateway, die actieve sessies altijd kan observeren.

**Door mappen bladeren** opent de ingebouwde mapbrowser van de Plaats-kiezer, ondersteund door de alleen-voor-beheerdersmethode `fs.listDir` en beperkt tot de geselecteerde Gateway of node. De Gateway en nodes die bladeren ondersteunen, vermelden hun bestandssysteem; een node die uitvoering ondersteunt maar geen `fs.listDir` heeft, accepteert nog steeds een handmatig ingevoerd absoluut pad. Recente plaatsen kunnen een map en de bijbehorende node samen herstellen zonder paden tussen hosts over te nemen. Bij verzenden wordt `sessions.create` aangeroepen met het eerste bericht, zodat de uitvoering tijdens dezelfde heen-en-terugreis begint en de UI naar de chat van de nieuwe sessie springt. Als de Gateway de sessie aanmaakt maar de eerste verzending afwijst, bewaart de chat de prompt en fout na herladen; met **Opnieuw proberen** wordt het bericht via de al aangemaakte sessie verzonden in plaats van nog een sessie aan te maken.

Binnen **Instellingen** bevat de eigen zijbalk **Vraag OpenClaw** en begint deze met een veld **Instellingen doorzoeken** om snel instellingensecties te vinden.

Op het desktopweb bevat een vast bedieningscluster linksboven in het inhoudsgebied — de webtegenhanger van de macOS-titelbalkstrook — de schakelaar om de zijbalk in te klappen (⌘B) en de zoekknop voor het opdrachtenpalet (⌘K). Als je bovenaan de zijbalk op de identiteitsrij van de agent klikt, wordt het agentmenu geopend; **Start** opent de hoofdsessie. Wanneer iets aandacht vereist — mislukte of achterstallige Cron-taken, modelauthenticatie die binnenkort verloopt of al is verlopen — verschijnen compacte aandachtschips boven de voettekst van de zijbalk waarmee je naar de bijbehorende pagina kunt doorklikken. De identiteitsrij toont de avatar van de agent (identiteitsafbeelding of emoji), de naam, de verbindingsstip en een live ondertitel. Het agentspecifieke menu bevat de inline agentwisselaar (voor opstellingen met meerdere agents), **Nieuwe agent**, "Wat kan deze agent?", en **Agentinstellingen**. Bij overzichten met meer dan tien agents verschijnt een filterveld en worden vastgemaakte agents eerst weergegeven; maak agents vast of los via de instellingenpagina Agents, waarbij de vastgemaakte set in het browserprofiel wordt opgeslagen. Als je een agent kiest, worden Chat plus Gebruik, Automatiseringen, Taken, Werkbord en Sessies tot die agent beperkt. Elke pagina met zo'n bereik bevat een bedieningselement **Agent** met **Alle agents** als uitweg; hiermee wordt het bereik van de gedeelde pagina verbreed zonder de concrete chatagent te wijzigen, terwijl directe sessielinks nog steeds hun doelsessie openen. De instellingenpagina Agents behoudt een eigen `?agent=`-selectie en volgt het gedeelde paginabereik niet. De voettekst is één identiteitskaart over de volledige breedte die offline beschikbaar blijft en **Opnieuw verbinden…** onder de laatst bekende accountnaam toont. Deze opent het app-/accountmenu, waarin de profielidentiteitskop wordt gevolgd door **Instellingen**, **Gebruik**, koppeling met mobiele apparaten, **Download de apps**, **Help** (help, Discord, Documentatie en het wijzigingslogboek), indien nodig een offline actie om het opnieuw te proberen, de versie-/buildchip en de schakelaar voor de kleurmodus. De buildchip opent de pagina Over. Wanneer de Gateway vanuit een broncheckout wordt uitgevoerd op een andere branch dan `main`, toont de voettekst die branchnaam ook in rood, zodat direct duidelijk is dat het om een Gateway gaat die niet voor een release bestemd is (bij release-installaties wordt deze nooit getoond). Shift-Command-Komma op Apple-platforms of Ctrl-Shift-Komma elders opent **Instellingen** zonder de gewone Command-Komma-sneltoets van de browser te overschrijven. Als je de zijbalk inklapt (⌘B of de schakelaar in het cluster), wordt deze volledig verborgen voor een werkruimte over de volledige breedte; wanneer de zijbalk is ingeklapt, behoudt het cluster linksboven de uitklapschakelaar en zoekfunctie en krijgt het een knop voor een nieuwe thread — overeenkomstig wat de macOS-app standaard in de titelbalk bevat. De zijbalk is op desktop het enige navigatiekader; er is geen bovenbalk. In smalle viewports wordt de zijbalk vervangen door een uitschuiflade achter een compacte koprij met de ladeschakelaar, het merk en de zoekfunctie voor het opdrachtenpalet; op telefoons neemt Chat die navigatierij op in de titelbalk, met de menu- en zoekbediening naast de sessietitel. In de macOS-app neemt de afzonderlijke koprij de vrije ruimte voor de titelbalk op in één compacte strook naast de vensterbediening. De navigatie gebruikt de gewone browsergeschiedenis, zodat je deze met de knoppen Vorige/Volgende van de browser kunt doorlopen; de macOS-app voegt naast de vensterbediening een systeemeigen zijbalkschakelaar en veegbewegingen op het trackpad toe, met knoppen Vorige/Volgende aan de rechterrand van de zijbalk wanneer deze is uitgeklapt en systeemeigen knoppen voor zoeken (opdrachtenpalet) en een nieuwe sessie wanneer deze is ingeklapt.

Openstaande goedkeuringen voegen ook een aandachtschip boven de voettekst van de zijbalk toe;
selecteer deze om de bijbehorende pagina Goedkeuringen te openen.

## Wat het kan (vandaag)

<AccordionGroup>
  <Accordion title="Chatten en praten">
    - Chat met het model via Gateway WS (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`). Bij gearchiveerde sessies blijft het invoerveld uitgeschakeld en wordt een banner met de actie **Dearchiveren** weergegeven voordat het gesprek kan worden voortgezet.
    - Bij het vernieuwen van de chatgeschiedenis wordt een begrensd recent venster aangevraagd met tekstlimieten per bericht, zodat grote sessies de browser niet dwingen een volledige transcriptpayload te renderen voordat de chat bruikbaar wordt.
    - Als je de muis boven een openbare GitHub-issue of link naar een pull request houdt of deze met het toetsenbord focust, worden de status, titel, auteur, recente activiteit, opmerkingen en wijzigingsstatistieken weergegeven. De verbonden Gateway haalt openbare metagegevens op en slaat deze in de cache zonder het linkdoel te wijzigen, ook wanneer de UI een externe Gateway gebruikt. De Gateway gebruikt `GH_TOKEN` of `GITHUB_TOKEN` wanneer beschikbaar, nadat is bevestigd dat de repository openbaar is; anders wordt de anonieme API van GitHub met een langere cache gebruikt.
    - Praat via realtime browsersessies. OpenAI gebruikt directe WebRTC, Google Live gebruikt een beperkt browsertoken voor eenmalig gebruik via WebSocket en realtime spraakplugins die alleen op de backend werken, gebruiken het relaytransport van de Gateway. Browsersessies met video-ondersteuning kunnen in Instellingen een apparaatlokale camera kiezen of vanuit het live voorbeeld van camera wisselen; de browser legt JPEG-frames vast voor de realtime provider zonder cameravideo via de Gateway te streamen. Sessies die door de client worden beheerd, starten met `talk.client.create`; Gateway-relaysessies starten met `talk.session.create`. De relay bewaart de inloggegevens van de provider op de Gateway terwijl de browser PCM van de microfoon via `talk.session.appendAudio` streamt, stuurt `openclaw_agent_consult`-toolaanroepen van de provider via `talk.client.toolCall` door voor Gateway-beleid en het grotere geconfigureerde OpenClaw-model, en routeert spraakbesturing van actieve uitvoeringen via `talk.client.steer` of `talk.session.steer`.
    - Stream toolaanroepen en live kaarten met tooluitvoer in Chat (agentgebeurtenissen). Toolactiviteit wordt weergegeven als rijen die zijn afgestemd op het type: shell-opdrachten tonen de opdracht met syntaxisaccentuering en uitvoer in terminalstijl; ondersteunde bewerkings- en schrijfaanroepen tonen begrensde inline verschillen, regelnummers indien beschikbaar en `+added -removed`-statistieken; en opeenvolgende aanroepen worden samengevouwen tot een samenvatting zoals "13 opdrachten uitgevoerd, 6 bestanden gelezen, 9 bestanden bewerkt". Terwijl een uitvoering actief is, bepaalt de nieuwste actieve aanroep de naam van de groepskop. Klap een rij uit om de resterende argumenten en onbewerkte uitvoer te bekijken.
    - Optionele door AI gegenereerde doeltitels voor complexe toolaanroepen (lange shell-opdrachten, plugintools met veel argumenten), ingeschakeld met `gateway.controlUi.toolTitles: true` (standaard uitgeschakeld). Titels zijn afkomstig van de gebatchte methode `chat.toolTitles` via de standaardroutering voor utilitymodellen — een expliciete `utilityModel` (door de beheerder gekozen provider, zoals bij andere utilitytaken), anders het opgegeven standaardmodel van de sessieprovider voor kleine modellen — en worden aan de Gateway-zijde per agent in de cache opgeslagen. Wanneer de opt-in is uitgeschakeld of er geen goedkoop model bruikbaar is, behouden de rijen hun deterministische labels en vindt er geen modelaanroep plaats.
    - Start of negeer tijdelijke, door het model voorgestelde vervolgtaken; geaccepteerde suggesties openen een nieuwe sessie in een beheerde worktree met de voorgestelde prompt.
    - Activiteitentabblad met browserlokale samenvattingen waarbij redactie vooropstaat, van live toolactiviteit uit bestaande levering van `session.tool`-/toolgebeurtenissen.

  </Accordion>
  <Accordion title="Kanalen, sessies, geheugen">
    - Kanalen: status van ingebouwde plus gebundelde/externe pluginkanalen, QR-aanmelding en configuratie per kanaal (`channels.status`, `web.login.*`, `config.patch`).
    - Bij het vernieuwen van kanaalcontroles blijft de vorige momentopname zichtbaar terwijl trage providercontroles worden voltooid, en worden gedeeltelijke momentopnamen gelabeld wanneer een controle of audit het UI-tijdsbudget overschrijdt.
    - Threads (een werkruimtepagina op `/sessions`, met daarnaast een tabblad **Worktrees**): geef standaard sessies van geconfigureerde agents weer, maak veelgebruikte sessies vast, wijzig hun naam, archiveer of herstel inactieve sessies, val terug vanaf verouderde sessiesleutels van niet-geconfigureerde agents en pas per sessie overschrijvingen toe voor model/denken/snel/uitgebreid/tracering/redenering (`sessions.list`, `sessions.patch`). Een drievoudig filter **Actief / Gearchiveerd / Alles** bestuurt zowel deze pagina als de zijbalk; Alles maakt gearchiveerde rijen minder prominent en voorziet ze van een expliciet label. Gearchiveerde sessies behouden hun transcripten, worden nooit automatisch opgeschoond en blijven opgeborgen totdat ze expliciet worden gedearchiveerd of verwijderd. Rijen tonen een ongelezen stip voor actieve sessies met activiteit sinds ze voor het laatst zijn gelezen, met acties om ze als ongelezen/gelezen te markeren (`sessions.patch { unread }`), en een actie Fork waarmee het transcript naar een nieuwe sessie wordt vertakt (`sessions.create { parentSessionKey, fork: true }`). Overzichtstegels boven de tabel vatten het geladen overzicht samen (aantal sessies, actieve uitvoeringen, ongelezen sessies, totaal aantal tokens en het aantal gearchiveerde sessies indien beschikbaar), elke rij bevat een typepictogram met een stip voor een actieve uitvoering, de status wordt weergegeven als een gewone stip plus label en de kolom Tokens toont een gebruiksmeter voor het contextvenster wanneer de sessie token- en contextgroottes rapporteert. Beheeracties voor rijen staan in een menu per rij (knop met drie verticale punten of rechtsklikken) dat het sessiemenu van de zijbalk weerspiegelt, en de rijlade toont de agentruntime en uitvoeringsduur naast de overige sessiedetails.
    - Systeemeigen Claude- en Codex-catalogi in de zijbalk streamen één host tegelijk en worden vervolgens opnieuw afgestemd na wijzigingen in de Node-connectiviteit, wanneer de pagina focus krijgt en hoogstens elke 30 seconden zolang deze zichtbaar is. Wijzigingen in de catalogus activeren sneller een extra ronde, zodat sessies die in de systeemeigen tools zijn gemaakt zonder herladen in de Control UI verschijnen. Rijen van Claude Desktop behouden ook hun lokale aangepaste groepslabel als dat aanwezig is; OpenClaw leest die toewijzing uit de lokale opslag van Desktop en schrijft er nooit naar.
    - Sessiegroepering: met het bedieningselement Groeperen op wordt de sessietabel ingedeeld in secties op basis van aangepaste groepen, kanaal, type, agent of datum. Aangepaste groepen blijven per sessie bewaard via `sessions.patch` (`category`), zodat ook sessies die vanuit berichtkanalen (Discord, Telegram, WhatsApp, ...) zijn gestart, kunnen worden gecategoriseerd; wijs groepen toe door rijen naar een sectie te slepen of met de groepsselector per rij, en maak groepen met de actie Nieuwe groep.
    - Geheugen (een tabblad op de pagina Agents, beperkt tot de geselecteerde agent): Dreaming-status, schakelaar voor in-/uitschakelen en lezer voor het Droomdagboek (`doctor.memory.status`, `doctor.memory.dreamDiary`, `config.patch`).
    - Geheugen importeren (`/memory-import`, bereikbaar via het tabblad Geheugen op de pagina Agents): bekijk een voorbeeld van en kopieer lokaal automatisch geheugen van Claude Code, geconsolideerd Codex-geheugen of Hermes-geheugenbestanden naar de werkruimte van de geselecteerde agent (`migrations.memory.plan`, `migrations.memory.apply`).
    - Geheugenaanbod tijdens onboarding: wanneer de Control UI in de onboardingmodus wordt geopend (`?onboarding=1`, gebruikt door de Linux-begeleidende app na de eerste installatie), biedt een dialoogvenster van één pagina aan om gedetecteerde geheugens te importeren met dezelfde plan-/toepassingsflow; bij overslaan blijft de instellingenpagina beschikbaar als later toegangspunt.

  </Accordion>
  <Accordion title="Cron, taken, plugins, skills, apparaten, uitvoeringsgoedkeuringen">
    - Automatiseringen (cron-taken): statistiekkaarten (aantal automatiseringen, aantal mislukte automatiseringen, plannerstatus, volgende activering) boven een tabschakelaar Automatiseringen/Uitvoeringsgeschiedenis; het tabblad Automatiseringen vermeldt taken in een filterbare tabel (Alle/Actief/Gepauzeerd, zoeken, filters voor planning en laatste uitvoering, actiemenu per rij) met daaronder startsuggesties, en het tabblad Uitvoeringsgeschiedenis toont recente uitvoeringen van alle automatiseringen (`cron.*`).
    - Taken: live register van actieve en recente achtergrondtaken met gekoppelde sessies en annulering (`tasks.*`). De zijbalk Achtergrondtaken van Chat groepeert actief en voltooid werk; selecteer een rij om de afgebakende prompt en uitvoer of het foutoverzicht te bekijken.
    - Plugins: bekijk de geïnstalleerde inventaris en samengestelde winkel, doorzoek ClawHub, installeer en verwijder plugincode en schakel geïnstalleerde plugins in of uit (`plugins.*`); rijen voor MCP-servers bewerken `mcp.servers` via de configuratiemethoden.
    - Skills: status, inschakelen/uitschakelen, installeren, API-sleutels bijwerken (`skills.*`).
    - Apparaten: één inventaris combineert records van gekoppelde apparaten, de nodecatalogus en live aanwezigheid (`device.pair.list`, `node.list`, `system-presence`). De Gateway-host staat vast bovenaan; gekoppelde clients tonen verbindingsstatus, rollen, tokens, mogelijkheden en opdrachten. Dubbele koppelingen worden samengevoegd tot een uitvouwbare groep en **N verouderde items opschonen** verwijdert in bulk door een beheerder bevestigde offline duplicaten die automatisch zijn goedgekeurd (stil lokaal, vertrouwde CIDR of via SSH geverifieerd) of dateren van vóór de herkomstregistratie van goedkeuringen. Vermeldingen kunnen worden verwijderd (`node.pair.remove`, `device.pair.remove`), apparaatkoppeling en hergoedkeuringen van nodes worden inline afgehandeld (`device.pair.*`, `node.pair.approve`/`reject`) en mobiele installatiecodes worden vanaf dezelfde kaart gemaakt.
    - Uitvoeringsgoedkeuringen: bewerk Gateway- of node-toestaanlijsten en het vraagbeleid voor `exec host=gateway/node` (`exec.approvals.*`).

  </Accordion>
  <Accordion title="Configuratie">
    - Bekijk/bewerk `~/.openclaw/openclaw.json` (`config.get`, `config.set`).
    - De navigatie door Instellingen begint met OpenClaw vragen en groepeert pagina's vervolgens op aandachtspunt: Algemeen, Uiterlijk en Meldingen bovenaan; Verbindingen (Verbinding, Kanalen, Communicatie, Apparaten); Agents en hulpmiddelen (Agents, AI en agents, Modelproviders, MCP, Automatisering, Labs); Privacy en beveiliging (Beveiliging, Goedkeuringen); en Systeem (Infrastructuur, Geavanceerd, Foutopsporing, Logboeken, Over). Algemeen is een compacte centrale pagina met standaardinstellingen voor modellen, taal en statistieken van de Gateway-host; elke andere instelling staat op precies één pagina.
    - Privacy en beveiliging: samengestelde rijen voor Gateway-authenticatie, uitvoeringsbeleid, browseractivering, hulpmiddelenprofiel, apparaatauthenticatie en mobiele koppeling, boven de door het schema ondersteunde secties `security`/`approvals`.
    - Goedkeuringen bevat een geschiedenis van 30 dagen, nieuwste eerst, voor afgehandelde uitvoerings-, plugin- en systeemagentverzoeken. Filter op soort of blader door oudere rijen om de beslissing, reden, bronsessie en door de Gateway vastgelegde toeschrijving van de afhandelaar te bekijken.
    - Labs biedt uitgebrachte experimentele schakelaars. Codemodus en Swarm zijn de huidige vermeldingen en slaan `tools.codeMode.enabled` en `tools.swarm.enabled` onmiddellijk op; niet-uitgebrachte experimenten verschijnen niet en schrijven geen speculatieve configuratiesleutels.
    - Meldingen: status van webpushmeldingen in de browser, abonneren/afmelden en een testverzending.
    - Geavanceerd: elke configuratiesectie zonder een samengestelde eigen pagina, plus de onbewerkte JSON5-editor (voorheen de geavanceerde modus van de pagina Algemeen).
    - Modelinstelling (`/settings/model-setup`) is een subpagina van Modelproviders, geopend vanuit de kop ervan.
    - Agents: een instellingenpagina (**Instellingen → Agents**, `/settings/agents`) met tabbladen per agent (Overzicht, Bestanden, Hulpmiddelen, Skills, Kanalen, Automatiseringen, Geheugen). Het tabblad Overzicht bewerkt de identiteit van de agent — weergavenaam, emoji en een avatarafbeelding die in de browser wordt verkleind en in grootte wordt begrensd vóór `agents.update`. Bij opslaan worden de geconfigureerde identiteitsvelden opgeslagen en gespiegeld naar de werkruimte `IDENTITY.md`; geconfigureerde waarden hebben voorrang op handmatige bewerkingen van dezelfde bestandsvelden.
    - Profiel: een instellingenpagina die de identiteit van de standaardagent toont met gebruiksstatistieken over de volledige periode — totaal aantal tokens, piekdag, langste sessie, activiteitsreeksen, een tokenheatmap over een jaar, meest gebruikte hulpmiddelen en kanaalhoogtepunten (`usage.cost`, `sessions.usage`).
    - MCP heeft een speciale instellingenpagina met serverrijen (transport, inschakeling, OAuth-/filter-/paralleliteitsoverzichten), directe bedieningselementen voor toevoegen/inschakelen/uitschakelen/verwijderen, veelgebruikte beheerdersopdrachten en de configuratie-editor met bereik `mcp`. De pagina Plugins blijft de centrale plek voor connectors met één klik en ontdekking.
    - Modelproviders: een instellingenpagina met elke geconfigureerde modelprovider, inclusief het merkpictogram, de authenticatiestatus (`models.authStatus`), modelbeschikbaarheid (`models.list`), live abonnements-/quota-/factureringsgegevens wanneer de provider die rapporteert (`usage.status`) en lokale sessie-uitgaven van de afgelopen 30 dagen (`sessions.usage`). De actie Vernieuwen leest de referentiestatus en het providergebruik opnieuw.
    - Verbinding: een instellingenpagina (onder **Verbindingen**) die de eigen Gateway-koppeling van het dashboard beheert — WebSocket-URL, Gateway-token, wachtwoord en standaardsessiesleutel — plus de meest recente momentopname van de handshake (status, bedrijfstijd, tikinterval, laatste vernieuwing van kanalen). De offline aanmeldpoort verwerkt de situatie zonder verbinding; deze pagina bewerkt de verbinding terwijl er verbinding is.
    - Toepassen en opnieuw starten met validatie (`config.apply`) en vervolgens de laatst actieve sessie activeren.
    - Schrijfbewerkingen bevatten een basishashbeveiliging om te voorkomen dat gelijktijdige bewerkingen worden overschreven.
    - Schrijfbewerkingen (`config.set`/`config.apply`/`config.patch`) controleren vooraf de actieve SecretRef-resolutie voor verwijzingen in de ingediende configuratiepayload; niet-opgeloste actieve ingediende verwijzingen worden vóór het schrijven geweigerd.
    - Bij het opslaan van formulieren worden verouderde geredigeerde tijdelijke aanduidingen verwijderd die niet vanuit de opgeslagen configuratie kunnen worden hersteld, terwijl geredigeerde waarden die nog aan opgeslagen geheimen zijn gekoppeld behouden blijven.
    - Schema- en formulierweergave zijn afkomstig van `config.schema` / `config.schema.lookup`, inclusief veld-`title`/`description`, overeenkomende UI-aanwijzingen, directe onderliggende samenvattingen, documentatiemetagegevens op geneste object-/jokerteken-/array-/compositieknooppunten, plus plugin- en kanaalschema's wanneer beschikbaar. De onbewerkte JSON-editor is alleen beschikbaar wanneer de momentopname veilig onbewerkt heen en terug kan worden geconverteerd; anders dwingt de Control UI de formuliermodus af.
    - Met "Herstellen naar opgeslagen" in de onbewerkte JSON-editor blijft de onbewerkt geschreven vorm behouden (opmaak, opmerkingen, indeling van `$include`) in plaats van een afgevlakte momentopname opnieuw weer te geven, zodat externe bewerkingen een herstelbewerking overleven wanneer de momentopname veilig heen en terug kan worden geconverteerd.
    - Gestructureerde SecretRef-objectwaarden worden alleen-lezen weergegeven in tekstinvoervelden van formulieren om onbedoelde beschadiging door conversie van object naar tekenreeks te voorkomen.

  </Accordion>
  <Accordion title="Gebruik">
    - Analyse van tokens en geschatte kosten op basis van sessies blijft gescheiden van providerfacturering.
    - Providerkaarten roepen `usage.status` aan en tonen live abonnementsnamen, quotavensters, saldi, uitgaven en budgetten die door geconfigureerde providerplugins worden gerapporteerd.
    - Een fout bij het ophalen van providergebruik blokkeert het dashboard voor sessies/kosten niet; niet-beschikbare providerkaarten tonen hun eigen foutstatus.

  </Accordion>
  <Accordion title="Foutopsporing, logboeken, bijwerken">
    - Foutopsporing: momentopnamen van status/gezondheid/modellen, gebeurtenislogboek en handmatige RPC-aanroepen (`status`, `health`, `models.list`).
    - Het gebeurtenislogboek bevat timinggegevens voor vernieuwings-/RPC-acties van de Control UI, timinggegevens voor trage chat-/configuratieweergave en vermeldingen over browserresponsiviteit voor lange animatieframes of langdurige taken wanneer de browser die PerformanceObserver-vermeldingstypen beschikbaar stelt.
    - Logboeken: live weergave van het einde van Gateway-logboekbestanden met filter/export (`logs.tail`).
    - Bijwerken: voer een pakket-/git-update plus herstart uit (`update.run`) met een herstartrapport en peil vervolgens `update.status` na het opnieuw verbinden om de actieve Gateway-versie te verifiëren.

  </Accordion>
  <Accordion title="Opmerkingen bij het automatiseringenpaneel">
    - Als je een rij selecteert, wordt een detailweergave op volledige pagina geopend met een schakelaar Actief/Gepauzeerd en Nu uitvoeren in de kop (uitvoeren indien gepland, klonen en verwijderen in het menu); het tabblad Instellingen bewerkt de automatisering inline (prompt, details, frequentie, geavanceerde overschrijvingen) en het tabblad Uitvoeringsgeschiedenis toont de uitvoeringen van die automatisering.
    - Startautomatiseringen onder de tabel vullen het aanmaakformulier vooraf in met een bewerkbare prompt en planning.
    - Voor geïsoleerde taken is de standaardbezorging een aankondigingssamenvatting; schakel over naar geen voor uitvoeringen die uitsluitend intern zijn.
    - Velden voor kanaal/doel verschijnen wanneer aankondigen is geselecteerd.
    - De Webhook-modus gebruikt `delivery.mode = "webhook"` waarbij `delivery.to` is ingesteld op een geldige HTTP(S)-Webhook-URL.
    - Voor taken in de hoofdsessie zijn de bezorgingsmodi Webhook en geen beschikbaar.
    - Geavanceerde bedieningselementen voor bewerken omvatten verwijderen na uitvoering, agentoverschrijving wissen, opties voor exacte/gespreide cron-uitvoering, overschrijvingen voor agentmodel/denkmodus en schakelaars voor bezorging op basis van beste inspanning.
    - Formuliervalidatie vindt inline plaats met fouten op veldniveau; bij ongeldige waarden blijft de knop Opslaan uitgeschakeld totdat deze zijn gecorrigeerd.
    - Stel `cron.webhookToken` in om een speciaal bearertoken te verzenden; als dit wordt weggelaten, wordt de Webhook zonder authenticatieheader verzonden.
    - `cron.webhook` is een ingetrokken verouderde terugvaloptie die door de huidige configuratievalidatie wordt geweigerd. Voer `openclaw doctor --fix` uit om opgeslagen taken die nog `notify: true` gebruiken, te migreren naar expliciete bezorging per taak via Webhook of bij voltooiing en verwijder de oude sleutel.

  </Accordion>
</AccordionGroup>

## Assistentgeheugen importeren

Open **Instellingen** → **Geheugen importeren** om lokaal geheugen van Codex of Claude Code
naar een OpenClaw-agent over te brengen. De Gateway ontdekt ondersteund lokaal geheugen zelf
op zijn host, zodat een externe Control UI importeert vanaf de Gateway-computer en niet vanaf de
browsercomputer.

1. Kies de bestemmingsagent.
2. Controleer de gedetecteerde broncollecties en Markdown-bestandsnamen. De bestandsinhoud
   wordt niet in de planrespons verzonden of op de pagina weergegeven.
3. Selecteer de collecties die je wilt importeren en bevestig. Bij toepassen wordt het plan opnieuw opgebouwd vóór
   het schrijven, zodat verouderde selecties veilig mislukken.
4. Als bestanden al bestaan, schakel je **Bestaande imports vervangen** in, vernieuw je het
   voorbeeld en bevestig je de vervanging.

Codex importeert alleen zijn geconsolideerde `MEMORY.md` en `memory_summary.md`. Claude
Code importeert Markdown uit mappen voor automatisch projectgeheugen en een geconfigureerde
`autoMemoryDirectory`; via deze pagina importeert het geen sessies, instellingen, instructies of
referenties. Bestanden worden gekopieerd onder `memory/imports/` in de
geselecteerde werkruimte, waar de actieve geheugenplugin ze kan indexeren. Bronnen worden
nooit gewijzigd.

Voor plannen en toepassen is `operator.admin` vereist. Elke toepassing maakt een geverifieerde
OpenClaw-back-up wanneer er statusgegevens bestaan, schrijft een geredigeerd migratierapport en bewaart
back-ups per item voordat bestaande doelbestanden worden vervangen. Zie
[Geheugenoverzicht](/nl/concepts/memory#import-from-coding-assistants) voor paden en
oproepgedrag.

## MCP-pagina

De speciale MCP-pagina is een beheerdersweergave voor door OpenClaw beheerde MCP-servers onder `mcp.servers`. De pagina start MCP-transporten niet zelf; gebruik haar om opgeslagen configuratie te bekijken en te bewerken en gebruik vervolgens `openclaw mcp doctor --probe` wanneer je live bewijs van de server nodig hebt.

Typische workflow:

1. Open **MCP** vanuit de zijbalk.
2. Bekijk de overzichtskaarten voor het totale aantal servers en het aantal ingeschakelde, OAuth- en gefilterde servers.
3. Controleer elke serverrij op transport, inschakeling, authenticatie, filters, time-outs en opdrachthints.
4. Voeg servers rechtstreeks op de MCP-pagina toe, schakel ze in of uit, of verwijder ze. Kies expliciet Streamable HTTP, SSE of stdio; stdio-opdrachtregels accepteren argumenten tussen aanhalingstekens, zoals paden met spaties. Gebruik de pagina **Plugins** voor connectors met één klik en detectie.
5. Bewerk de relevante `mcp`-configuratiesectie voor geavanceerde servervelden, zoals omgevingsvariabelen, werkmappen, headers, TLS-/mTLS-paden, OAuth-metadata, toolfilters en Codex-projectiemetadata.
6. Gebruik **Save** om de configuratie te schrijven, of **Save & Publish** wanneer de actieve Gateway de gewijzigde configuratie moet toepassen.
7. Voer `openclaw mcp status --verbose`, `openclaw mcp doctor --probe` of `openclaw mcp reload` uit vanuit een terminal voor statische diagnostiek, live bewijs of het verwijderen van de gecachte runtime.

De pagina maskeert URL-achtige waarden die referentiegegevens bevatten voordat deze worden weergegeven en zet servernamen tussen aanhalingstekens in opdrachtfragmenten, zodat gekopieerde opdrachten ook met spaties of shell-metatekens blijven werken. Volledige CLI- en configuratiereferentie: [MCP](/nl/cli/mcp).

## Tabblad Activiteit

Het tabblad Activiteit bevindt zich in **Instellingen › Systeem**, naast Logs en Debug. Het is een tijdelijke, browserlokale waarnemer van live toolactiviteit, afgeleid van dezelfde Gateway-`session.tool`-/toolgebeurtenissenstroom die de toolkaarten van Chat aanstuurt. Het voegt geen extra Gateway-gebeurtenisfamilie, eindpunt, duurzaam activiteitenarchief, metriekfeed of externe waarnemersstroom toe.

Activiteitsitems bewaren alleen opgeschoonde samenvattingen en gemaskeerde, ingekorte uitvoervoorbeelden. Waarden van toolargumenten worden niet opgeslagen in de activiteitsstatus; de UI geeft aan dat argumenten verborgen zijn en registreert alleen het aantal argumentvelden. De lijst in het geheugen is gekoppeld aan het huidige browsertabblad, blijft behouden tijdens navigatie binnen de Control UI en wordt opnieuw ingesteld wanneer de pagina opnieuw wordt geladen, van sessie wordt gewisseld of **Wissen** wordt gebruikt.

## Operatorterminal

De koppelbare operatorterminal is standaard uitgeschakeld. Stel `gateway.terminal.enabled: true` in en start de Gateway opnieuw om deze in te schakelen. De terminal vereist een `operator.admin`-verbinding en opent een host-PTY in de werkruimte van de actieve agent. Nieuwe tabbladen volgen de momenteel geselecteerde chatagent.

<Warning>
De terminal is een onbeperkte hostshell en neemt de omgeving van het Gateway-proces over. Schakel deze alleen in voor vertrouwde operatorimplementaties. OpenClaw weigert terminalsessies voor agents met `sandbox.mode: "all"`; als een actieve agent naar die modus wordt gewijzigd, worden de bestaande en lopende terminalsessies ervan gesloten.
</Warning>

Gebruik **Ctrl + backtick** om het dock te tonen of verbergen. De indeling ondersteunt koppeling aan de onder- en rechterkant, past zich aan de browserviewport aan en behoudt meerdere shelltabbladen. Zie [Gateway-configuratie](/nl/gateway/configuration-reference#gateway) voor `gateway.terminal.enabled` en de optionele `gateway.terminal.shell`-overschrijving.

Door de eigenaar geautoriseerde agents zonder sandbox kunnen de tool `terminal` gebruiken voor langdurig of interactief werk dat de operator moet volgen. Elke toolaanroep kan de eigen Gateway-PTY's van de agent openen, lezen, beschrijven, van formaat wijzigen, sluiten of weergeven. Nieuwe sessies openen standaard een gezamenlijk gekoppeld tabblad in de Control UI, zodat de agent en operator de uitvoer delen en beiden kunnen typen of het formaat kunnen wijzigen. Agenttoegang is beperkt tot de exacte sessie: een agent kan geen door de operator aangemaakte terminals of terminals die door een andere agentsessie zijn geopend lezen of beheren.

Sleep een of meer bestanden naar de actieve terminal of gebruik de paperclipknop om bestanden te kiezen. OpenClaw plaatst elk bestand tijdelijk op de machine die eigenaar is van de PTY en plakt de absolute, voor de shell geciteerde paden bij de cursor; het drukt nooit op Enter en voert de invoer nooit uit. Een compacte batchindicator toont het huidige bestand en het aantal voltooide bestanden. Annuleren stopt de resterende batch zonder paden te plakken; een mislukte overdracht blijft zichtbaar, zodat je vanaf dat bestand opnieuw kunt proberen zonder voltooide bestanden opnieuw te uploaden. Afbeeldingen, pdf's, archieven en andere bestandstypen worden geaccepteerd tot 16 MiB per bestand. Tijdelijk geplaatste bestanden gebruiken een persoonlijke tijdelijke systeemmap op POSIX-hosts (mapmodus `0700`, bestandsmodus `0600`) of een map binnen de ACL-grens van het gebruikersprofiel op Windows, plus een opschoningstimer van 24 uur. Verplaats of kopieer daarom alles wat je wilt bewaren.

Het invoegen van paden ondersteunt PowerShell, `cmd.exe` en herkende POSIX-shells (`sh`, Bash, Dash, Ash, Ksh, Zsh en Fish), inclusief Git Bash op Windows. Andere shelloverschrijvingen worden geweigerd omdat hun regels voor citeren niet veilig kunnen worden afgeleid; voer de Gateway uit binnen WSL voor een systeemeigen WSL-terminal en Linux-uploadpaden. `cmd.exe`-paden die `%` of `!` bevatten, worden ook geweigerd omdat die shell deze tekens zelfs binnen dubbele aanhalingstekens uitbreidt.

Codex- en Claude Code-sessies die in de sessiezijbalk worden gevonden, kunnen in hun systeemeigen CLI binnen hetzelfde terminalpaneel worden geopend. Stel in **Instellingen › Chat** de optie **Open Codex/Claude threads in** in op **Terminal** om met een normale klik op een rij `codex resume` of `claude --resume` te openen; standaard blijft de alleen-lezen OpenClaw-viewer actief. Het contextmenu of kebabmenu van een rij biedt altijd beide keuzes en de viewerheader bevat **Open in terminal** wanneer die sessie daarvoor in aanmerking komt.

Geschiktheid wordt per sessie en per host bepaald. Gateway-lokale sessies starten de hervattingsopdracht van de provider op de Gateway-host. Sessies op gekoppelde nodes starten een toegestane provideropdracht op de node die eigenaar is en sturen alleen de uitvoer-, invoer- en formaatwijzigingsgebeurtenissen van die PTY door; hierdoor wordt geen algemene nodeshell blootgesteld en worden geen door de browser aangeleverde opdrachten geaccepteerd. Bestandsuploads gebruiken de afzonderlijke, in grootte beperkte node-opdracht `terminal.upload` en blijven gekoppeld aan de reeds geopende terminalsessie. Keur de upgrade van de nodekoppeling goed wanneer die opdracht voor het eerst verschijnt. Nodes die de bijbehorende opdracht voor het hervatten van de terminal niet aanbieden, waaronder ingebedde workerbridges zonder duplexstreaming, houden de viewer beschikbaar en geven aan dat het openen van de terminal niet beschikbaar is; oudere nodes kunnen nog steeds een terminal uitvoeren, maar kunnen geen gesleepte bestanden ontvangen.

Sessies die eigendom zijn van een verbinding blijven behouden na verbrekingen: bij het opnieuw laden van een pagina, de slaapstand van een laptop of een korte netwerkstoring wordt de sessie op de Gateway losgekoppeld in plaats van beëindigd, en hetzelfde browsertabblad wordt bij het opnieuw verbinden weer gekoppeld, waarbij recente uitvoer opnieuw wordt afgespeeld. Losgekoppelde sessies die eigendom zijn van een verbinding worden na `gateway.terminal.detachedSessionTimeoutSeconds` beëindigd (standaard 300 seconden; `0` herstelt beëindiging bij verbreking). Het koppelen aan een van deze sessies blijft een overname in tmux-stijl.

Sessies die eigendom zijn van een agent zijn niet aan een browserverbinding gebonden. `terminal.attach` voegt elke browser als viewer toe zonder het eigendom over te nemen, en bij het sluiten van een viewertabblad wordt alleen die browser losgekoppeld. De PTY blijft bestaan totdat de agent die eigenaar is deze sluit, het proces eindigt, beleid deze uitschakelt of de Gateway wordt afgesloten. `terminal.list` markeert elk item als eigendom van een verbinding of agent, en met `terminal.text` kan een beheerdersverbinding recente platte-tekstuitvoer lezen zonder te koppelen.

De terminal is ook beschikbaar als een schermvullend document met alleen de terminal op `/?view=terminal`. De iOS- en Android-apps sluiten deze pagina in hun Terminal-schermen in en hergebruiken de opgeslagen Gateway-referentiegegevens; de beschikbaarheid volgt dezelfde `gateway.terminal.enabled`- en `operator.admin`-beperking en de pagina toont een melding wanneer de verbonden Gateway de terminal niet aanbiedt.

## Browserpaneel

De Control UI bevat een koppelbaar browserpaneel dat de door de Gateway beheerde browser (dezelfde die agents aansturen via de [browsertool](/nl/tools/browser-control)) in elke gewone webbrowser weergeeft — er is geen systeemeigen webview vereist. Het verschijnt wanneer de verbonden Gateway `browser.request` aanbiedt aan een `operator.admin`-verbinding; met de wereldbolknop in de werkruimtebalk van de thread kun je het tonen of verbergen. Het paneel toont een live momentopname van de pagina met tabbladen, een bewerkbare URL-balk, terug/vooruit/opnieuw laden en openen in je browser, kan rechts of onderaan worden gekoppeld en stuurt klikken, scrollen met het wiel en eenvoudige tekstinvoer door naar de externe pagina.

Twee vastlegmodi bundelen paginacontext voor de agent:

- **Annoteren (potlood)**: teken vrije markeringen over de pagina. **Naar chat verzenden** voegt de lijnen samen met de schermafbeelding, voegt de afbeelding toe aan het actieve chatopstelveld en vult vooraf een prompt in die de pagina-URL, titel en elk gemarkeerd gebied beschrijft, zodat de agent precies weet wat je hebt omcirkeld.
- **Inspecteren (aanwijzer)**: beweeg de cursor om het onderliggende element te bekijken (selector, toegankelijke naam, rol, grootte); klik om de details van dat element en een gemarkeerde schermafbeelding via dezelfde opstelstroom te verzenden. Voor inspecteren, scrollen met het wiel en terug/vooruit is `browser.evaluateEnabled` vereist (standaard ingeschakeld).

De macOS-app behoudt de systeemeigen zijbalk van de linkbrowser voor links waarop in het dashboard wordt geklikt; het browserpaneel werkt daar ook en is op elk ander platform de manier om pagina's te annoteren.

## Chatgedrag

<AccordionGroup>
  <Accordion title="Semantiek van verzenden en geschiedenis">
    - `chat.send` is **niet-blokkerend**: er wordt onmiddellijk een bevestiging met `{ runId, status: "started" }` verzonden en het antwoord wordt gestreamd via `chat`-gebeurtenissen. Vertrouwde Control UI-clients kunnen ook optionele timingmetadata voor bevestigingen ontvangen voor lokale diagnostiek.
    - Chatuploads accepteren afbeeldingen en niet-videobestanden. Afbeeldingen behouden het oorspronkelijke afbeeldingspad; andere bestanden worden opgeslagen als beheerde media en in de geschiedenis weergegeven als bijlagelinks.
    - Opnieuw verzenden met dezelfde `idempotencyKey` retourneert tijdens de uitvoering `{ status: "in_flight" }` en na voltooiing `{ status: "ok" }`.
    - Antwoorden van `chat.history` hebben voor de veiligheid van de UI een maximale grootte. Wanneer transcriptvermeldingen te groot zijn, kan de Gateway lange tekstvelden afkappen, zware metadatablokken weglaten en te grote berichten vervangen door een tijdelijke aanduiding (`[chat.history omitted: message too large]`).
    - Wanneer een zichtbaar assistentbericht in `chat.history` is afgekapt, kan de zijlezer op aanvraag de volledige, voor weergave genormaliseerde transcriptvermelding ophalen via `chat.message.get`, op basis van `sessionKey`, zo nodig de actieve `agentId` en transcript-`messageId`. Als de Gateway nog steeds niet meer kan retourneren, toont de lezer een expliciete status ‘niet beschikbaar’ in plaats van stilzwijgend het afgekorte voorbeeld te herhalen.
    - Door de assistent gemaakte of gegenereerde afbeeldingen worden bewaard als verwijzingen naar beheerde media en opnieuw aangeboden via geauthenticeerde media-URL's van de Gateway, zodat opnieuw laden niet afhankelijk is van onbewerkte base64-afbeeldingspayloads die in het antwoord met de chatgeschiedenis blijven staan.
    - Bij het weergeven van `chat.history` verwijdert de Control UI inline-richtlijntags die uitsluitend voor weergave dienen uit zichtbare assistenttekst (bijvoorbeeld `[[reply_to_*]]` en `[[audio_as_voice]]`), XML-payloads voor toolaanroepen in platte tekst (waaronder `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` en afgekorte toolaanroepblokken) en uitgelekte ASCII-/volledige-breedtecontroletokens van het model. Assistentvermeldingen waarvan de volledige zichtbare tekst uitsluitend bestaat uit het exacte stille token `NO_REPLY` / `no_reply` of het Heartbeat-bevestigingstoken `HEARTBEAT_OK` worden weggelaten.
    - Tijdens een actieve verzending en de uiteindelijke vernieuwing van de geschiedenis houdt de chatweergave lokale optimistische gebruikers-/assistentberichten zichtbaar als `chat.history` kortstondig een oudere momentopname retourneert; het canonieke transcript vervangt die lokale berichten zodra de Gateway-geschiedenis is bijgewerkt.
    - Live `chat`-gebeurtenissen vertegenwoordigen de afleveringsstatus, terwijl `chat.history` opnieuw wordt opgebouwd vanuit het duurzame sessietranscript. Na tool-final-gebeurtenissen laadt de Control UI de geschiedenis opnieuw en voegt alleen een kleine optimistische staart samen; de transcriptgrens wordt beschreven in [WebChat](/nl/web/webchat).
    - `chat.inject` voegt een assistentnotitie toe aan het sessietranscript en zendt een `chat`-gebeurtenis uit voor updates die uitsluitend voor de UI bestemd zijn (geen agentuitvoering, geen aflevering via een kanaal).
    - De zijbalk vermeldt elke geladen actieve sessie per agentsectie en in de categorieën vastgezet/kanaal/werk/aangepast/Chats, met één actie Nieuwe sessie waarmee het conceptvenster wordt geopend. Bij het openen van een zichtbare rij wordt alleen de markering verplaatst. Sessies kunnen op Vastgezet worden neergezet om ze vast te zetten, of op een aangepaste groep of Chats om ze te verplaatsen; aangepaste groepen kunnen worden ingeklapt en via slepen opnieuw worden gerangschikt, groepsnamen en de volgorde worden via de Gateway gesynchroniseerd en de ingeklapte status blijft in de browser behouden. Een nieuwe dashboardsessie krijgt asynchroon een beknopte gegenereerde titel op basis van het eerste bericht dat geen opdracht is; expliciete namen en de identiteit van de geauthenticeerde afzender blijven gescheiden, zodat accountnamen nooit als gegenereerde titels worden gebruikt. Stel `agents.defaults.utilityModel` (of `agents.entries.*.utilityModel`) in om deze afzonderlijke modelaanroep naar een goedkoper model te routeren; als dat afzonderlijke model mislukt, wordt de titelgeneratie eenmaal opnieuw geprobeerd met het primaire model. Door een andere agentsectie uit te vouwen, kun je door de sessies van die agent bladeren zonder de geopende chat te verlaten.
    - Zoeken in threads bevindt zich in het opdrachtenpalet (⌘K of de zoekknop in de besturingsgroep linksboven): wanneer je een zoekopdracht typt, wordt een begrensd aantal pagina's met overeenkomsten voor verschillende agents doorzocht, worden interne onderliggende/cron-rijen gefilterd en worden zichtbare overeenkomsten naast navigatieopdrachten vermeld. De pagina Threads behoudt de volledige doorzoekbare lijst met filters.
    - Elke zijbalkrij biedt directe toegang tot vastzetten en een volledig contextmenu voor de ongelezen status, hernoemen, afsplitsen, groeperen, archiveren en verwijderen. Voor meervoudig geselecteerde rijen (Cmd/Ctrl-klik, Shift-klik voor bereiken) verschijnt een batchmenu voor de ongelezen status, groeperen, archiveren en verwijderen; archiveren/verwijderen als batch blijft uitgeschakeld tenzij elke geselecteerde sessie kan worden gearchiveerd. Een actieve uitvoering en de hoofdsessie van een agent kunnen niet worden gearchiveerd. Als de momenteel geselecteerde sessie wordt gearchiveerd of verwijderd, schakelt Chat terug naar de hoofdsessie van die agent.
    - In de macOS-app gebruikt het OpenClaw-beeldmerk de anders lege strook van de systeemeigen titelbalk naast de vensterknoppen, in plaats van een rij in de zijbalk in te nemen.
    - Bij desktopbreedtes blijven de chatbedieningselementen op één compacte rij staan en worden ze ingeklapt tijdens het omlaag scrollen door het transcript; omhoog scrollen, terugkeren naar de bovenkant of de onderkant bereiken herstelt de bedieningselementen.
    - De sessiekop toont naast de werkruimtechip een kleine stapel profielfoto's wanneer andere personen dezelfde sessie bekijken; deze toont maximaal vier avatars van kijkers met een aantal voor de overige kijkers en verdwijnt wanneer je alleen bent.
    - Opeenvolgende dubbele berichten met uitsluitend tekst worden weergegeven als één tekstballon met een aantalbadge. Berichten met afbeeldingen, bijlagen, tooluitvoer of Canvas-voorbeelden worden niet samengevoegd.
    - Gebruikersberichtballonnen bevatten transcriptacties: een terugspoelknop die bij aanwijzen verschijnt (bevestigingspop-over met de optie "Don't ask again"), plus bij rechtsklikken **Terugspoelen tot hier** en **Vanaf hier afsplitsen**. Terugspoelen verwijst de sessie opnieuw naar de status vlak vóór dat bericht en zet de tekst ervan terug in het invoerveld om deze te bewerken en opnieuw te verzenden (`sessions.rewind`, `operator.admin`); afsplitsen maakt een nieuwe sessie op basis van het actieve padprefix vóór het bericht, opent deze en vult het invoerveld met dezelfde tekst (`sessions.fork`, `operator.write`). Beide acties worden uitgeschakeld met een verklarende tooltip terwijl de agent bezig is, gelden alleen voor bewaarde gebruikersberichten en worden geweigerd voor sessies waarvan het gesprek eigendom is van een extern agentharnas. Terugspoelen verplaatst alleen de chatcontext — bestanden en andere neveneffecten van tools worden niet teruggedraaid — en het transcript van vóór het terugspoelen blijft behouden in de alleen-toevoegenopslag van de sessie. Wanneer die opslag meerdere transcripttakken bevat, toont de chattitelbalk een takmenu met voor elke tak het meest recente bericht, het aantal berichten en de recentheid; door een inactieve tak te selecteren, schakelt de huidige sessie terug naar dat bewaarde pad (`sessions.branches.list`, `operator.read`; `sessions.branches.switch`, `operator.admin`). Schakelen tussen takken is eveneens niet beschikbaar terwijl de agent bezig is en het selecteren van de reeds actieve tak geeft aan de RPC-grens een getypeerde no-op-fout. De afzonderlijke verbergactie voor gebruikersballonnen verbergt een bericht alleen in de huidige browser; het bericht blijft in het transcript staan en de agent blijft het zien.
    - Wanneer de checkout van een sessie zich op een niet-standaardtak van een GitHub-repository bevindt, zet de chatweergave pull-requestchips vast boven het invoerveld: PR-nummer, repository, tak, aantallen verschillen, een CI-badge en de status concept/samengevoegd/gesloten, elk met een koppeling naar de PR. De rij toont maximaal twee chips — live-PR's (open/concept) eerst — en met de knop "Show more" wordt de ingeklapte geschiedenis van samengevoegde/gesloten PR's zichtbaar. De CI-badge opent een kleine CI-bewakingspop-over met aantallen geslaagde/mislukte/actieve/overgeslagen controles en een koppeling naar de controlepagina van de PR. Detectie vindt aan de serverzijde plaats via `controlUi.sessionPullRequests`, waarbij de `GH_TOKEN`/`GITHUB_TOKEN` van de Gateway opnieuw worden gebruikt wanneer deze zijn ingesteld. Wanneer de snelheidslimiet van de GitHub-API wordt bereikt, behouden chips de laatst bekende status en tonen ze een waarschuwing dat de status mogelijk verouderd is; als je een chip sluit, wordt deze voor die sessie verborgen in het huidige browserprofiel. Voordat er een PR bestaat, toont de rij de tak zelf — repository, taknaam en de +/−-grootte van het verschil ten opzichte van de samenvoegbasis van de standaardtak (vastgelegde en niet-vastgelegde wijzigingen). Zodra de gepushte tak commits bevat die kunnen worden vergeleken, voegt de rij een knop Create PR toe die de pagina voor een nieuwe pull request van GitHub opent; daarvoor krijgt een sessie met gewijzigde bestanden (vastgelegd, niet-vastgelegd of niet-gevolgd) nog steeds de rij, maar zonder de knop. De rij verbergt zichzelf zolang er een open PR of concept-PR bestaat. De takrij is uitsluitend afkomstig uit de lokale Git-gegevens en blijft dus beschikbaar wanneer GitHub een snelheidslimiet toepast; de rij toont dezelfde waarschuwing voor een verouderde status, omdat "geen PR gevonden" niet kan worden vertrouwd totdat de limiet opnieuw is ingesteld.
    - Het sessieverschillenpaneel toont wat de checkout van een sessie daadwerkelijk heeft gewijzigd: de takknop in de werkruimtebalk of de chattitelbalk opent het detailpaneel met per bestand de verschillen van de tak en van niet-vastgelegd en niet-gevolgd werk ten opzichte van de samenvoegbasis van de standaardtak van de checkout — statusstip, hernoempijl, +/−-aantallen per bestand, inklapbare bestanden en markeringen "N ongewijzigde regels" tussen wijzigingsblokken. Verschillen worden aan de serverzijde berekend via de Gateway-methode `sessions.diff` (`operator.read`-bereik); binaire en te grote bestanden worden teruggebracht tot vermeldingen met alleen statistieken en de knop verschijnt alleen wanneer de verbonden Gateway `sessions.diff` aankondigt.
    - Elk Chat-deelvenster heeft een titelbalk. Klik op de sessietitel om deze te hernoemen; de werkruimtechip kopieert het checkoutpad of de tak en kan lokale Gateway-werkruimten weergeven in het bestandsbeheer van de host. Sessies op afstand en exec-node-sessies behouden de kopieeracties, maar verbergen de weergaveactie.
    - De threadwerkruimtebalk in elk Chat-deelvenster vermeldt threadbestanden, projectbestanden en artefacten. De balk is standaard aan de rechterrand van het deelvenster vastgezet; sleep de kop ervan (of gebruik de vastzetknop) om de balk naar de onderkant te verplaatsen. De keuze wordt opgeslagen in het huidige browserprofiel. Een ingeklapte balk neemt helemaal geen ruimte in: open deze opnieuw met ⇧⌘B of met de bestandsschakelaar in de titelbalk, die een badge met het aantal gewijzigde bestanden bevat. Het afzonderlijke detailpaneel voor bestanden, tools en Canvas blijft ongewijzigd.
    - Als je klikt op een bestandsverwijzing in de chat, een bestandspad in een uitgevouwen toolkaart voor lezen/bewerken/schrijven of een bestandsrij in de werkruimtebalk, wordt het bestandsdetailpaneel geopend: een op CodeMirror gebaseerde codeweergave met syntaxismarkering, regelnummers, springen naar een regel, zoeken in het bestand, kopieeracties en een menu om het bestand in een externe editor te openen. Wanneer de Gateway `sessions.files.set` aankondigt aan een `operator.admin`-verbinding, voegt het paneel een bewerkingsmodus toe met bijhouden van wijzigingen en opslaan via Cmd/Ctrl-S; niet-opgeslagen concepten blijven tijdens navigatie tussen bestanden, panelen en sessies in het huidige browsertabblad behouden totdat ze expliciet worden opgeslagen of verwijderd. Opslaan gebeurt met vergelijken-en-verwisselen op basis van een inhoudshash die wordt geretourneerd door `sessions.files.get`: als het bestand op schijf is gewijzigd sinds het werd geladen (bijvoorbeeld omdat de agent doorwerkte), toont het paneel een conflictmelding met de acties Reload (de nieuwste inhoud gebruiken) en Overwrite (de lokale bewerking behouden). Schrijfbewerkingen verlopen via dezelfde fs-veilige werkruimtecontroles als leesbewerkingen — padinsluiting, weigering van symbolische/harde koppelingen en een limiet van 256 KB voor UTF-8 — en overschrijven alleen bestaande bestanden; de editor maakt of verwijdert nooit bestanden.
    - De balk met achtergrondtaken in elk Chat-deelvenster vermeldt de achtergrondtaken en subagents van de huidige agent (`tasks.list` met bereik per agent, live gehouden door `task`-gebeurtenissen): actief werk toont een live timer voor de verstreken tijd, het aantal toolgebruiksmomenten, de momenteel gebruikte tool en een stopknop; de inklapbare sectie met voltooide taken voegt uitvoeringsduren toe; en de koppeling Transcript bekijken opent de onderliggende sessie van de taak in het deelvenster. Open de balk met de activiteitsschakelaar in de titelbalk; de taakmomentopname wordt vooraf geladen en toont daardoor een badge met het aantal actieve taken zonder dat de balk eerst hoeft te worden geopend. De pagina Taken blijft het volledige overzicht voor alle agents.
    - De werkruimterail, de rail voor achtergrondtaken en het detailpaneel passen zich aan de eigen breedte van elk deelvenster aan in plaats van aan die van het venster: in een smal deelvenster of compact venster worden beide rails als stroken onderaan weergegeven (besturingselementen voor vastzetten aan de zijkant blijven verborgen totdat het deelvenster breder wordt; de werkruimterail krijgt als eerste de zijpositie wanneer er slechts één kolom past), en het detailpaneel wordt onder de thread gestapeld met een horizontale greep om het formaat te wijzigen, in plaats van dezelfde rij ermee te delen. Op viewports ter grootte van een telefoon wordt het detailpaneel nog steeds op volledig scherm geopend.
    - De model- en denkmoduskeuzelijsten in de chatkop werken de actieve sessie onmiddellijk bij via `sessions.patch`; het zijn permanente sessieoverschrijvingen, geen verzendopties die slechts voor één beurt gelden.
    - **Gesplitste weergave:** open deze via de titelbalk van de chat (naast de schakelaars voor de threaddiff, achtergrondtaken en threadbestanden) en splits het actieve deelvenster vervolgens naar rechts of omlaag in zoveel deelvensters als er passen. Elk deelvenster heeft een eigen thread, transcript, composer en toolstream.
    - Agents met de tool `screen` kunnen dezelfde wijzigingen aan deelvensters, zijbalk, terminal, browser, focus en navigatie aanvragen terwijl een geschikte Control UI is verbonden. Protocol v1 past de opdracht toe op elke verbonden geschikte Control UI; zie [Scherm](/tools/screen).
    - Sleep een sessie vanuit de zijbalk naar de chat om deze in een deelvenster te openen. Een geanimeerd neerzetvoorbeeld beweegt vloeiend tussen zones en benoemt het resultaat — "Splitsen" boven precies de helft die een nieuw deelvenster zal innemen, "Hier openen" boven een volledig deelvenster — en neerzetten werkt ook in de modus met één deelvenster.
    - Het actieve gesplitste deelvenster bepaalt de selectie in de zijbalk en de URL. De titelbalk ervan bevat extra besturingselementen voor splitsen en sluiten; met scheidingslijnen kunnen kolommen en gestapelde deelvensters van formaat worden gewijzigd, en de browser slaat de indeling lokaal op zodat deze behouden blijft na herladen.
    - Op smalle schermen behoudt de gesplitste weergave de indeling, maar wordt alleen het actieve deelvenster weergegeven, inclusief de kop met het besturingselement voor sluiten.
    - Als je een bericht verzendt terwijl een wijziging in de modelkeuzelijst voor dezelfde sessie nog wordt opgeslagen, wacht de composer op die sessiepatch voordat `chat.send` wordt aangeroepen, zodat voor het verzenden het geselecteerde model wordt gebruikt.
    - Door `/new` te typen, wordt dezelfde nieuwe dashboardsessie gemaakt en geactiveerd als met Nieuwe chat, behalve wanneer `session.dmScope: "main"` is geconfigureerd en de huidige bovenliggende sessie de hoofdsessie van de agent is; in dat geval wordt de hoofdsessie ter plaatse opnieuw ingesteld. Door `/reset` te typen, blijft de expliciete herinitialisatie ter plaatse van de Gateway voor de huidige sessie behouden.
    - De modelkeuzelijst van de chat vraagt de geconfigureerde modelweergave van de Gateway op. Als `agents.defaults.modelPolicy.allow` niet leeg is, bepaalt dat beleid de keuzelijst, inclusief `provider/*`-vermeldingen die catalogi met een providerbereik dynamisch houden. Anders toont de keuzelijst geconfigureerde vermeldingen plus providers met bruikbare authenticatie; aliassen en instellingen onder `agents.defaults.models` beperken deze niet. De volledige catalogus blijft beschikbaar via de debug-RPC `models.list` met `view: "all"`.
    - Wanneer recente gebruiksrapporten van Gateway-sessies de huidige contexttokens bevatten, toont de werkbalk van de chatcomposer een kleine ring voor contextgebruik met het gebruikte percentage. Open de ring voor het huidige contextvenster, de tokenaantallen van de laatste uitvoering en de geschatte totale kosten, de provider-/modelidentiteit en, indien gerapporteerd, de uitsplitsing van de invoer-, uitvoer- en cachekosten van het meest recente providerantwoord. Bij hoge contextdruk krijgt de ring een waarschuwingsstijl en bij aanbevolen Compaction-niveaus toont deze een compacte knop waarmee het normale Compaction-pad voor de sessie wordt uitgevoerd. Verouderde tokensnapshots worden verborgen totdat de Gateway opnieuw actueel gebruik rapporteert.

  </Accordion>
  <Accordion title="Praatmodus (realtime in de browser)">
    De Praatmodus gebruikt een geregistreerde realtime-spraakprovider. Configureer OpenAI met `talk.realtime.provider: "openai"` plus een API-sleutelprofiel voor `openai`, `talk.realtime.providers.openai.apiKey` of `OPENAI_API_KEY`. OpenAI Realtime gebruikt de openbare Platform-API en vereist een Platform-API-sleutel; aanmelden via Codex OAuth volstaat niet voor dit oppervlak. Configureer Google met `talk.realtime.provider: "google"` plus `talk.realtime.providers.google.apiKey`. De browser ontvangt nooit een standaard-API-sleutel van de provider: OpenAI ontvangt een tijdelijk Realtime-clientgeheim voor WebRTC, en Google Live ontvangt een eenmalig, beperkt Live API-authenticatietoken voor een WebSocket-sessie in de browser, waarbij instructies en tooldeclaraties door de Gateway in het token zijn vergrendeld. Providers die alleen een realtime-backendbridge aanbieden, werken via het relaytransport van de Gateway, zodat referenties en sockets van de leverancier aan de serverzijde blijven terwijl browseraudio via geauthenticeerde Gateway-RPC's wordt verzonden. De prompt voor de Realtime-sessie wordt door de Gateway samengesteld; `talk.client.create` accepteert geen door de aanroeper opgegeven overschrijvingen van instructies.

    Permanente standaardwaarden voor provider, model, stem, transport, redeneerinspanning, exacte VAD-drempel, stilteduur en voorlooppadding staan in **Instellingen → Communicatie → Praten**; om ze te wijzigen is toegang tot `operator.admin` vereist. Als je Gateway-relay configureert, wordt het backend-relaypad afgedwongen; bij configuratie van WebRTC blijft de sessie eigendom van de client en mislukt deze in plaats van stilzwijgend terug te vallen op relay als de provider geen browsersessie kan maken.

    Het bedieningselement voor Praten is de microfoonknop in de werkbalk van het invoerveld. Het pijltje toont **Systeemstandaard** en elke microfoon die de browser beschikbaar stelt, waaronder USB-, Bluetooth- en virtuele ingangen. De geselecteerde apparaat-ID blijft lokaal in de browser en wordt nooit naar de Gateway verzonden; als dat exacte apparaat verdwijnt, vraagt Praten je een andere ingang te kiezen in plaats van stilzwijgend via een andere microfoon op te nemen. Terwijl Praten actief is, verandert de microfoonknop in een pilvormige knop met de live-invoerniveaumeter; klikken stopt de spraakinvoer en bij aanwijzen verschijnt het stoppictogram. Schermlezers kondigen `Connecting voice input...`, `Listening...` of `Asking OpenClaw...` aan terwijl een realtime-toolaanroep via `talk.client.toolCall` het geconfigureerde grotere model raadpleegt. Het stoppen van een actief agentantwoord blijft een afzonderlijk vierkant bedieningselement **Stoppen** naast de pilvormige knop.

    **Videopraten** is beschikbaar voor browsersessies met OpenAI Realtime WebRTC en Google Live. Klik op de cameraknop, sta toegang tot camera en microfoon toe en bevestig de lokale voorvertoning. OpenAI verzendt één begrensd JPEG-frame via het browserdatakanaal wanneer `describe_view` om visuele context vraagt. Google Live verzendt begrensde JPEG-frames rechtstreeks vanuit de browser naar de provider met het ondersteunde maximum van één frame per seconde en beantwoordt functieaanroepen van `describe_view` met de status van de camerastream. Cameraframes gaan nooit via de Gateway. Als je Praten stopt, wordt de voorvertoning gesloten en worden beide mediasporen vrijgegeven. Zie de [mogelijkheden van de Live API](https://ai.google.dev/gemini-api/docs/live-api/capabilities#video) en de [handleiding voor functieaanroepen](https://ai.google.dev/gemini-api/docs/live-api/tools) van Google voor de verbindingscontracten van de provider.

    Live-smoketest voor beheerders: `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` verifieert de OpenAI-backendbridge via WebSocket, de WebRTC-SDP-uitwisseling van OpenAI in de browser, de browserconfiguratie van Google Live met beperkt token, een JPEG-frame en een functierondgang via `describe_view`, en de Gateway-relayadapter voor de browser met gesimuleerde microfoonmedia. De opdracht toont alleen de providerstatus en registreert geen geheimen.

  </Accordion>
  <Accordion title="Stoppen en afbreken">
    - Klik op **Stoppen**. Uitvoeringen met een exacte lokale uitvoerings-ID roepen `chat.abort` aan; wanneer de status van de geselecteerde sessie actief werk meldt maar de Control UI geen lokale uitvoerings-ID heeft, roept deze in plaats daarvan `sessions.abort` aan. Voor niet-globale sessies verwijdert dat pad voor de geselecteerde sessie ook vervolgberichten uit de wachtrij, zodat die het werk na het stoppen niet opnieuw kunnen starten.
    - Terwijl een uitvoering actief is, gebruiken normale vervolgberichten de effectieve `messages.queue`-modus van de Gateway. `steer` voegt ze in de actieve beurt in; andere modi behouden de duurzame bezorging via de wachtrij van de browser. Als bijsturing wordt geweigerd, wordt ook op die wachtrij teruggevallen. Klik op **Bijsturen** bij een bericht in de wachtrij om het handmatig in te voegen.
    - **Instellingen → Vormgeving → Chat → Vervolgberichten terwijl de agent werkt** kan die serverstandaard voor de huidige browser overschrijven. De pagina markeert een overschrijving expliciet en biedt **Terugzetten naar serverstandaard**. `Steer into the active run` verzendt vervolgberichten onmiddellijk, terwijl `Queue until the run ends` ze vasthoudt totdat de uitvoering is voltooid.
    - Typ `/stop` (of afzonderlijke afbreekzinnen zoals `stop`, `stop action`, `stop run`, `stop openclaw`, `please stop`) om buiten de normale gegevensstroom af te breken.
    - `chat.abort` ondersteunt `{ sessionKey }` (geen `runId`) om alle actieve uitvoeringen voor die sessie af te breken. De Control UI gebruikt `sessions.abort` wanneer deze geen lokale uitvoerings-ID heeft.

  </Accordion>
  <Accordion title="Behoud van gedeeltelijke uitvoer bij afbreken">
    - Wanneer een uitvoering wordt afgebroken, kan gedeeltelijke tekst van de assistent nog steeds in de UI worden weergegeven.
    - De Gateway slaat afgebroken gedeeltelijke assistenttekst op in de transcriptgeschiedenis wanneer gebufferde uitvoer bestaat.
    - Opgeslagen vermeldingen bevatten afbreekmetadata, zodat transcriptgebruikers gedeeltelijke uitvoer door afbreken kunnen onderscheiden van uitvoer na normale voltooiing.

  </Accordion>
</AccordionGroup>

## Verbindingsverlies en opnieuw verbinden

Zodra een sessie tot stand is gebracht, word je bij een verbroken Gateway-verbinding niet afgemeld. Het dashboard
blijft zichtbaar met een zwevende amberkleurige pil met "Gateway-verbinding verbroken — Opnieuw verbinden…" onder de bovenste
balk, terwijl de client automatisch opnieuw probeert met oplopende wachttijd (800 ms tot 15 s). Live-updates en
realtime-/sessieacties worden gepauzeerd totdat de verbinding terugkeert; **Nu opnieuw proberen** in de pil dwingt een
onmiddellijke poging af. De chat blijft bewerkbaar: gewone tekst en verzonden bijlagen worden bewaard in de
Gateway-/sessiegebonden browseropslag van het huidige tabblad, weergegeven als wachtend op opnieuw verbinden en
automatisch verzonden wanneer de Gateway terugkeert. Live-bedieningselementen en slashopdrachten blijven niet beschikbaar terwijl
je offline bent, behalve dat **Stoppen** een exacte lokale uitvoerings-ID in de wachtrij kan plaatsen om opnieuw af te spelen. Een stopactie die alleen voor een sessie geldt,
wordt niet opnieuw afgespeeld, omdat er mogelijk nieuwer werk in die sessie start voordat de verbinding terugkeert.

Wanneer deze browser al referenties bevat (een geconfigureerd token/wachtwoord of een goedgekeurd apparaat-
token), tonen de eerste opening en het opnieuw laden een klein geanimeerd OpenClaw-logo terwijl de verbinding
tot stand wordt gebracht, in plaats van kort het aanmeldscherm te tonen. Het aanmeldscherm verschijnt alleen wanneer er nog geen referenties
zijn opgeslagen of wanneer de Gateway ze actief weigert (ongeldig token/wachtwoord, ingetrokken koppeling) —
situaties die jouw invoer vereisen in plaats van wachten.

## PWA-installatie en webpush

De Control UI wordt geleverd met een `manifest.webmanifest` en een serviceworker, zodat moderne browsers deze als zelfstandige PWA kunnen installeren. Met Web Push kan de Gateway de geïnstalleerde PWA via meldingen activeren, zelfs wanneer het tabblad of browservenster niet geopend is.

In de macOS-app toont de instellingenpagina voor meldingen de systeemeigen meldingstoestemming van de app in plaats van browserpush, omdat de app meldingen systeemeigen aflevert.

Als de pagina direct na een OpenClaw-update **Protocol komt niet overeen** toont, open je eerst het dashboard opnieuw met `openclaw dashboard` en voer je een harde vernieuwing uit. Als het probleem aanhoudt, wis je de sitegegevens voor de oorsprong van het dashboard of test je in een privébrowservenster; een oud tabblad of de serviceworkercache van de browser kan een Control UI-bundel van vóór de update blijven uitvoeren met de nieuwere Gateway.

| Oppervlak                                          | Functie                                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `ui/public/manifest.webmanifest`                   | PWA-manifest. Browsers bieden "App installeren" aan zodra het bereikbaar is. |
| `ui/public/sw.js`                                  | Serviceworker die `push`-gebeurtenissen en klikken op meldingen verwerkt.    |
| `state/openclaw.sqlite` → `web_push_vapid_keys`    | Automatisch gegenereerd VAPID-sleutelpaar waarmee Web Push-payloads worden ondertekend.   |
| `state/openclaw.sqlite` → `web_push_subscriptions` | Opgeslagen browserabonnementseindpunten, sleutels en registratietijdstempels. |

Upgrades vanuit de buiten gebruik gestelde `push/vapid-keys.json`- en `push/web-push-subscriptions.json`-opslag worden geïmporteerd door `openclaw doctor --fix`. Stop de Gateway voordat je die reparatie uitvoert, zodat een ouder proces tijdens het importeren geen buiten gebruik gestelde status opnieuw kan maken. Voer de reparatie uit voordat je Web Push na een upgrade gebruikt; registratie, bezorging, verwijdering en sleutelomzetting weigeren door te gaan zolang een buiten gebruik gestelde bron of een onderbroken Doctor-claim aanwezig blijft. De Gateway-runtime leest en schrijft uitsluitend SQLite.

Overschrijf het VAPID-sleutelpaar via omgevingsvariabelen in het Gateway-proces wanneer je sleutels wilt vastzetten (implementaties met meerdere hosts, rotatie van geheimen of tests):

- `OPENCLAW_VAPID_PUBLIC_KEY`
- `OPENCLAW_VAPID_PRIVATE_KEY`
- `OPENCLAW_VAPID_SUBJECT` (standaardwaarde is `https://openclaw.ai`)

De Control UI gebruikt deze bereikgebonden Gateway-methoden om browserabonnementen te registreren en te testen:

- `push.web.vapidPublicKey` haalt de actieve openbare VAPID-sleutel op.
- `push.web.subscribe` registreert een `endpoint` plus `keys.p256dh`/`keys.auth`.
- `push.web.unsubscribe` verwijdert een geregistreerd eindpunt.
- `push.web.test` verzendt een testmelding naar het abonnement van de aanroeper.

<Note>
Web Push staat los van het iOS APNS-relaypad (zie [Configuratie](/nl/gateway/configuration) voor push via relay) en de methode `push.test`, die is gericht op systeemeigen mobiele koppeling.
</Note>

## Gehoste insluitingen

Assistentberichten kunnen gehoste webinhoud inline weergeven met de shortcode `[embed ...]`. Het iframe-sandboxbeleid wordt beheerd door `gateway.controlUi.embedSandbox`:

De kern-tool [`show_widget`](/nl/tools/show-widget) geeft zelfstandige SVG of HTML rechtstreeks vanuit een toolaanroep weer. De browser en ondersteunde systeemeigen chatclients adverteren de Gateway-mogelijkheid `inline-widgets`, en het resulterende Canvas-document blijft beschikbaar wanneer de chatgeschiedenis opnieuw wordt geladen. Discord Activities biedt dezelfde toolnaam op Discord; uitvoeringen die vanuit andere kanalen afkomstig zijn, ontvangen deze niet.

<Tabs>
  <Tab title="strikt">
    Schakelt scriptuitvoering binnen gehoste insluitingen uit.
  </Tab>
  <Tab title="scripts (standaard)">
    Staat interactieve insluitingen toe met behoud van oorsprongsisolatie; doorgaans voldoende voor zelfstandige browserspellen/widgets.
  </Tab>
  <Tab title="vertrouwd">
    Voegt `allow-same-origin` toe boven op `allow-scripts` voor documenten van dezelfde site die bewust uitgebreidere bevoegdheden nodig hebben.
  </Tab>
</Tabs>

```json5
{
  gateway: {
    controlUi: {
      embedSandbox: "scripts",
    },
  },
}
```

<Warning>
Gebruik `trusted` alleen wanneer het ingesloten document daadwerkelijk gedrag van dezelfde oorsprong nodig heeft. Voor de meeste door agents gegenereerde spellen en interactieve canvassen is `scripts` de veiligere keuze.
</Warning>

Absolute externe insluit-URL's van `http(s)` blijven standaard geblokkeerd. Stel `gateway.controlUi.allowExternalEmbedUrls: true` in om toe te staan dat `[embed url="https://..."]` pagina's van derden laadt.

## Indeling van het chattranscript

Het chattranscript gebruikt een gecentreerd, goed leesbaar kader dat is uitgelijnd met het invoerveld. Uitvoer van de assistent en tools blijft links uitgelijnd, terwijl je eigen berichten binnen dat kader rechts uitgelijnd blijven. In sessies met meerdere gebruikers (bijvoorbeeld een groepschat die vanuit een kanaalplugin wordt doorgestuurd) worden berichten van andere geïdentificeerde deelnemers links uitgelijnd weergegeven met de avatar en naam van de auteur en een stabiele kleur per identiteit, zodat alleen de berichten van de aangemelde gebruiker als 'van mij' worden weergegeven. Wanneer er twee of meer geïdentificeerde deelnemers aanwezig zijn, bevatten antwoorden van de assistent een kleine markering 'Antwoord aan naam' met de naam van de deelnemer wiens bericht de beurt activeerde. Systeemvermeldingen, zoals lokale uitvoer van slash-opdrachten, worden weergegeven als gecentreerde meldingsrijen zonder avatar.

## Breedte van chatberichten

Gebruikers met brede monitoren kunnen de transcriptbreedte aanpassen onder **Instellingen → Chat →
Berichtbreedte**. De voorkeur blijft opgeslagen in de lokale opslag van die browser. Ondersteunde
vormen zijn onder meer gewone lengtes en percentages, zoals `960px` of `82%`, plus
begrensde breedte-expressies met `min(...)`, `max(...)`, `clamp(...)`, `calc(...)` en
`fit-content(...)`.

## Tailnet-toegang (aanbevolen)

<Tabs>
  <Tab title="Geïntegreerde Tailscale Serve (voorkeur)">
    Houd de Gateway op loopback en laat Tailscale Serve deze via HTTPS proxyen:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Open `https://<magicdns>/` (of je geconfigureerde `gateway.controlUi.basePath`).

    Standaard kunnen Serve-verzoeken van de Control UI/WebSocket worden geauthenticeerd via Tailscale-identiteitsheaders (`tailscale-user-login`) wanneer `gateway.auth.allowTailscale` `true` is. OpenClaw verifieert de identiteit door het adres `x-forwarded-for` op te zoeken met `tailscale whois` en het met de header te vergelijken. OpenClaw accepteert deze alleen wanneer het verzoek loopback bereikt met de `x-forwarded-*`-headers van Tailscale. Voor operatorsessies van de Control UI met een browserapparaatidentiteit slaat dit geverifieerde Serve-pad ook de heen-en-terugstap voor apparaatkoppeling over; browsers zonder apparaatidentiteit en verbindingen met de noderol volgen nog steeds de normale apparaatcontroles. Stel `gateway.auth.allowTailscale: false` in als je expliciete gedeelde geheime referenties wilt vereisen, zelfs voor Serve-verkeer, en gebruik vervolgens `gateway.auth.mode: "token"` of `"password"`.

    Voor dat asynchrone Serve-identiteitspad worden mislukte authenticatiepogingen voor hetzelfde client-IP-adres en authenticatiebereik geserialiseerd voordat naar de snelheidslimiet wordt geschreven. Bij gelijktijdige onjuiste nieuwe pogingen vanuit dezelfde browser kan het tweede verzoek daarom `retry later` tonen, in plaats van dat twee gewone verschillen parallel wedijveren.

    <Warning>
    Serve-authenticatie zonder token veronderstelt dat de Gateway-host wordt vertrouwd. Als niet-vertrouwde lokale code op die host kan worden uitgevoerd, vereis dan authenticatie met een token of wachtwoord.
    </Warning>

  </Tab>
  <Tab title="Aan tailnet binden + token">
    ```bash
    openclaw gateway --bind tailnet --token "$(openssl rand -hex 32)"
    ```

    Open `http://<tailscale-ip>:18789/` (of je geconfigureerde `gateway.controlUi.basePath`).

    Plak het overeenkomende gedeelde geheim in de UI-instellingen (verzonden als `connect.params.auth.token` of `connect.params.auth.password`).

  </Tab>
</Tabs>

## Onbeveiligde HTTP

Als je het dashboard via gewone HTTP opent (`http://<lan-ip>` of `http://<tailscale-ip>`), wordt de browser in een **niet-beveiligde context** uitgevoerd en blokkeert deze WebCrypto. Standaard **blokkeert** OpenClaw Control UI-verbindingen zonder apparaatidentiteit.

De ondersteunde uitzondering zonder apparaatidentiteit is geslaagde Control UI-authenticatie voor operators
via `gateway.auth.mode: "trusted-proxy"`. Er bestaat geen permanente configuratieschakelaar
waarmee apparaatidentiteit wordt uitgeschakeld.

**Aanbevolen oplossing:** gebruik HTTPS (Tailscale Serve) of open de UI lokaal via `https://<magicdns>/` (Serve) of `http://127.0.0.1:18789/` (op de Gateway-host).

<AccordionGroup>
  <Accordion title="Opmerking over vertrouwde proxy's">
    - Geslaagde authenticatie via een vertrouwde proxy kan **operator**-sessies van de Control UI zonder apparaatidentiteit toelaten.
    - Dit geldt **niet** voor Control UI-sessies met de noderol.
    - Loopback-reverseproxy's op dezelfde host voldoen nog steeds niet aan authenticatie via een vertrouwde proxy; zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth).

  </Accordion>
</AccordionGroup>

Zie [Tailscale](/nl/gateway/tailscale) voor richtlijnen voor de HTTPS-configuratie.

## Contentbeveiligingsbeleid

De Control UI wordt geleverd met een streng `img-src`-beleid: alleen assets van **dezelfde origin**, `data:`-URL's en lokaal gegenereerde `blob:`-URL's zijn toegestaan. Externe `http(s)`- en protocolrelatieve afbeeldings-URL's worden door de browser geweigerd en leiden nooit tot netwerkverzoeken.

In de praktijk:

- Avatars en afbeeldingen die via relatieve paden worden aangeboden (bijvoorbeeld `/avatars/<id>`), worden nog steeds weergegeven, inclusief geauthenticeerde avatarroutes die de UI ophaalt en omzet in lokale `blob:`-URL's.
- Inline `data:image/...`-URL's worden nog steeds weergegeven.
- Lokale `blob:`-URL's die door de Control UI worden gemaakt, worden nog steeds weergegeven.
- Avatars voor GitHub-linkvoorbeelden worden door de Gateway opgehaald vanaf de vaste avatarhost van GitHub en teruggestuurd als begrensde `data:`-URL's; de browser van de operator maakt nooit verbinding met de externe avatarhost.
- Externe avatar-URL's uit kanaalmetadata worden door de avatarhelpers van de Control UI verwijderd en vervangen door het ingebouwde logo/de ingebouwde badge, zodat een gecompromitteerd of kwaadaardig kanaal geen willekeurige externe afbeeldingsverzoeken vanuit de browser van een operator kan afdwingen.

Dit is altijd ingeschakeld en niet configureerbaar.

## Authenticatie van de avatarroute

Wanneer Gateway-authenticatie is geconfigureerd, vereist het avatar-eindpunt van de Control UI hetzelfde Gateway-token als de rest van de API:

- `GET /avatar/<agentId>` retourneert de avatarafbeelding alleen aan geauthenticeerde aanroepers. `GET /avatar/<agentId>?meta=1` retourneert de avatarmetadata volgens dezelfde regel.
- Niet-geauthenticeerde verzoeken aan beide routes worden geweigerd (overeenkomstig de verwante route voor assistentmedia), zodat de avatarroute de agentidentiteit niet kan lekken op hosts die verder zijn beveiligd.
- De Control UI stuurt bij het ophalen van avatars het Gateway-token door als bearer-header en gebruikt geauthenticeerde blob-URL's, zodat de afbeelding nog steeds in dashboards wordt weergegeven.

Als je Gateway-authenticatie uitschakelt (niet aanbevolen op gedeelde hosts), wordt de avatarroute eveneens niet-geauthenticeerd, net als de rest van de Gateway.

## Authenticatie van de route voor assistentmedia

Wanneer Gateway-authenticatie is geconfigureerd, gebruiken lokale mediavoorbeelden van de assistent een route in twee stappen:

- `GET /__openclaw__/assistant-media?meta=1&source=<path>` vereist de normale Control UI-authenticatie voor operators; de browser verzendt het Gateway-token als bearer-header wanneer de beschikbaarheid wordt gecontroleerd.
- Geslaagde metadata-antwoorden bevatten een kortlevende `mediaTicket` die is beperkt tot precies dat bronpad.
- Door de browser weergegeven URL's voor afbeeldingen, audio, video en documenten gebruiken `mediaTicket=<ticket>` in plaats van het actieve Gateway-token of wachtwoord. Het ticket verloopt snel en kan geen andere bron autoriseren.

Hierdoor blijft mediaweergave compatibel met de systeemeigen media-elementen van browsers, zonder herbruikbare Gateway-referenties in zichtbare media-URL's te plaatsen.

## Goedkeuringslinks

Goedkeuringsmeldingen voor operators kunnen rechtstreeks linken naar een zelfstandig goedkeuringsdocument dat wordt aangeboden onder de gereserveerde naamruimte `${controlUiBasePath}/approve/{approvalId}` (bijvoorbeeld `/approve/<approvalId>`, of `/openclaw/approve/<approvalId>` met een geconfigureerd basispad). De URL blijft gedurende de levensduur van de goedkeuring stabiel en kan veilig tussen je eigen apparaten worden doorgestuurd: de URL identificeert de goedkeuring, maar autoriseert deze nooit.

- De naamruimte `/approve/<approvalId>` met één segment wordt door de Gateway vóór HTTP-routes van plugins gereserveerd voor **alle** HTTP-methoden, zodat een pluginroute een goedkeuringsdocument nooit kan overschaduwen of onderscheppen.
- Voor het openen van een goedkeuringsdocument is dezelfde Gateway-authenticatie vereist als voor de rest van de Control UI (token/wachtwoord, Tailscale Serve-identiteit of identiteit via een vertrouwde proxy); referenties maken nooit deel uit van de goedkeurings-URL.
- Wanneer het aanbieden van de Control UI is uitgeschakeld, retourneren verzoeken aan de naamruimte `404` in plaats van door te vallen naar pluginhandlers.
- Aanmelden in een goedkeuringsdocument is tijdelijk voor die pagina: dit overschrijft niet de Gateway-selectie of instellingen die door de volledige Control UI in dezelfde browser zijn opgeslagen.

De Gateway biedt statische bestanden aan vanuit `dist/control-ui`:

```bash
pnpm ui:build
```

Optionele absolute basis (vaste asset-URL's):

```bash
OPENCLAW_CONTROL_UI_BASE_PATH=/openclaw/ pnpm ui:build
```

Lokale ontwikkeling (afzonderlijke ontwikkelserver):

```bash
pnpm ui:dev
```

Laat de UI vervolgens verwijzen naar de WebSocket-URL van je Gateway (bijvoorbeeld `ws://127.0.0.1:18789`).

## Lege Control UI-pagina

Als de browser een leeg dashboard laadt en DevTools geen bruikbare fout toont, kan een extensie of vroeg contentscript hebben verhinderd dat de JavaScript-moduleapp werd geëvalueerd. De statische pagina bevat een eenvoudig HTML-herstelpaneel dat verschijnt wanneer `<openclaw-app>` na het opstarten niet is geregistreerd.

Gebruik de actie **Try again** van het paneel nadat je de browseromgeving hebt gewijzigd, of laad de pagina handmatig opnieuw na deze controles:

- Schakel extensies uit die in alle pagina's code injecteren, met name extensies met `<all_urls>`-contentscripts.
- Probeer een privévenster, een schoon browserprofiel of een andere browser.
- Laat de Gateway actief en controleer na de browserwijziging opnieuw dezelfde dashboard-URL.

## Fouten opsporen/testen: ontwikkelserver + externe Gateway

De Control UI bestaat uit statische bestanden; het WebSocket-doel is configureerbaar en kan afwijken van de HTTP-origin. Dit is handig wanneer je de Vite-ontwikkelserver lokaal wilt gebruiken, maar de Gateway elders wordt uitgevoerd.

<Steps>
  <Step title="Start de UI-ontwikkelserver">
    ```bash
    pnpm ui:dev
    ```
  </Step>
  <Step title="Openen met gatewayUrl">
    ```text
    http://localhost:5173/?gatewayUrl=ws%3A%2F%2F<gateway-host>%3A18789
    ```

    Optionele eenmalige authenticatie (indien nodig):

    ```text
    http://localhost:5173/?gatewayUrl=wss%3A%2F%2F<gateway-host>%3A18789#token=<gateway-token>
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Opmerkingen">
    - `gatewayUrl` wordt na het laden in localStorage opgeslagen en uit de URL verwijderd.
    - Als je via `gatewayUrl` een volledig `ws://`- of `wss://`-eindpunt doorgeeft, codeer de waarde dan als URL zodat de browser de querytekenreeks correct verwerkt.
    - `token` moet waar mogelijk via het URL-fragment (`#token=...`) worden doorgegeven. Fragmenten worden niet naar de server verzonden, waardoor lekken via verzoeklogboeken en de Referer worden voorkomen. Verouderde `?token=`-queryparameters worden voor compatibiliteit nog steeds één keer geïmporteerd, maar alleen als terugvaloptie, en worden direct na het opstarten verwijderd.
    - `password` wordt uitsluitend in het geheugen bewaard.
    - Wanneer `gatewayUrl` is ingesteld, valt de UI niet terug op referenties uit de configuratie of omgeving. Geef `token` (of `password`) expliciet op; ontbrekende expliciete referenties zijn een fout.
    - Gebruik `wss://` wanneer de Gateway zich achter TLS bevindt (Tailscale Serve, HTTPS-proxy enzovoort).
    - `gatewayUrl` wordt alleen geaccepteerd in een venster op het hoogste niveau (niet ingesloten) om clickjacking te voorkomen.
    - Openbare Control UI-implementaties buiten loopback moeten `gateway.controlUi.allowedOrigins` expliciet instellen (volledige origins). Privéloads vanaf hetzelfde origin via LAN/Tailnet vanaf loopback, RFC1918/link-local, `.local`, `.ts.net` of Tailscale CGNAT-hosts worden geaccepteerd zonder terugval op de Host-header in te schakelen.
    - Bij het opstarten kan de Gateway lokale origins zoals `http://localhost:<port>` en `http://127.0.0.1:<port>` invullen op basis van de effectieve runtimebinding en poort, maar externe browser-origins vereisen nog steeds expliciete vermeldingen.
    - Gebruik `gateway.controlUi.allowedOrigins: ["*"]` alleen voor strikt gecontroleerde lokale tests; dit betekent dat elke browser-origin wordt toegestaan, niet 'overeenkomen met de host die ik gebruik'.
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` schakelt de modus voor terugval op de Host-header voor origins in, maar dit is een gevaarlijke beveiligingsmodus.

  </Accordion>
</AccordionGroup>

```json5
{
  gateway: {
    controlUi: {
      allowedOrigins: ["http://localhost:5173"],
    },
  },
}
```

Details over het instellen van externe toegang: [Externe toegang](/nl/gateway/remote).

## Gerelateerd

- [Dashboard](/nl/web/dashboard) — Gateway-dashboard
- [Statuscontroles](/nl/gateway/health) — bewaking van de Gateway-status
- [TUI](/nl/web/tui) — terminalgebruikersinterface
- [WebChat](/nl/web/webchat) — browsergebaseerde chatinterface
