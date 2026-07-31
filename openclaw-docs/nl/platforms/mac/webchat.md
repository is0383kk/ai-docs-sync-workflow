---
read_when:
    - Foutopsporing voor de Mac WebChat-weergave of loopbackpoort
summary: Hoe de Mac-app de Gateway WebChat integreert en hoe je deze debugt
title: WebChat (macOS)
x-i18n:
    generated_at: "2026-07-27T05:55:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5e5983954e12d8546a01d089eda54e7eb0c60b4c92eff670f91797cd022c9fd
    source_path: platforms/mac/webchat.md
    workflow: 16
---

De macOS-menubalkapp integreert de WebChat-UI als een native SwiftUI-weergave. Deze maakt verbinding met de Gateway en gebruikt standaard de primaire sessie voor de geselecteerde agent (`main`, of `global` wanneer `session.scope` gelijk is aan `global`).

Het volledige chatvenster is een native gesplitste weergave:

- **Sessiezijbalk**: doorzoekbare sessielijst met vastgemaakte, door de Gateway ondersteunde groeps- en recente secties. Gestarte onderliggende sessies worden binnen elke sectie onder hun bovenliggende sessie genest; ingeklapte bovenliggende sessies vatten actieve, mislukte en ongelezen onderliggende sessies samen. Contextmenu's ondersteunen sessie-informatie, hernoemen, vastmaken, afsplitsen, gelezen/ongelezen, archiveren/herstellen, de sessiesleutel kopiëren en verwijderen. De primaire actie voor een nieuwe sessie (of Shift-Cmd-N) maakt deze onmiddellijk aan via `sessions.create`; in de aangrenzende pop-over met opties kan een agent worden geselecteerd en een beheerde worktree met een optionele basisreferentie worden aangevraagd.
- **Vensterwerkbalk**: ring voor contextgebruik (tokens en sessiekosten, met een compacte actie), modelbediening en een menu met sessieacties. Modellen worden per provider gegroepeerd, met de standaardprovider eerst, terwijl vastgemaakte en recente modellen bovenaan blijven staan. Met de bedieningselementen kan het denkniveau van het model worden overgenomen of overschreven, de uitgebreidheid van toolaanroepen worden gekozen en Fast responses worden in- of uitgeschakeld. Via het menu kan de huidige sessie worden hernoemd of afgesplitst en kan de status voor vastmaken, gelezen of archiveren worden bijgewerkt. **Sessies…** (Shift-Cmd-S) opent het beheer voor Actief/Gearchiveerd om in de Gateway te zoeken, groepen te beheren, sessies te inspecteren, te hernoemen, vast te maken, te archiveren en te herstellen. De selectiemodus past vastmaken, losmaken, archiveren of verwijderen toe op meerdere actieve sessies, terwijl afzonderlijke fouten zichtbaar blijven. Afzonderlijke vinkjes in het menu tonen of verbergen de redenering en toolactiviteit van de assistent; beide zijn standaard ingeschakeld en worden tussen starts onthouden.
- **Transcript en opsteller**: assistentberichten worden als platte tekst met een avatar weergegeven, gebruikersberichten als bubbels in de accentkleur. Openstaande vragen van de agent worden weergegeven als native kaarten met opties voor enkelvoudige of meervoudige selectie, vrije-tekstantwoorden via **Overig**, aftellingen tot vervaldatum en een gedeelde eindstatus. Lege chats bieden startprompts voor de desktop. Door `/` te typen, wordt automatisch aanvullen voor slash-opdrachten geopend, ondersteund door `commands.list`, met toetsenbordnavigatie via de pijltoetsen/Tab/Return/Escape. Klik met de rechtermuisknop op een bericht om de zichtbare Markdown zonder verborgen redenering te kopiëren. Afgekorte assistentberichten bieden ook **Volledig bericht openen**, waarmee een selecteerbare Markdown-lezer wordt geladen. Gebruik **Luisteren** voor TTS via de Gateway, met lokale spraak als terugvaloptie.
- **Spraakbediening**: de opsteller kan de bestaande macOS Talk Mode starten of stoppen zonder de menubalkoverlay ervan te vervangen. Terwijl Talk Mode actief is, toont de opsteller de status luisteren/denken/spreken, live audioactiviteit en een uitvouwbaar doorlopend transcript. Klik met de rechtermuisknop op de knop Talk om **System Default** of een aangesloten microfoon te kiezen; dit is dezelfde microfoonselectie die door Voice Wake en push-to-talk wordt gebruikt. Als een geselecteerde microfoon wordt losgekoppeld, valt de actieve Talk-sessie terug op de systeemstandaard en wordt de selectie opnieuw geprobeerd wanneer Talk Mode de volgende keer start. Met een afzonderlijke microfoonactie wordt een spraaknotitie opgenomen wanneer Talk Mode de audio-opname niet beheert.

Het verankerde compacte chatpaneel vanuit de menubalk behoudt de compacte indeling met één kolom en dezelfde bedieningselementen voor model, denken, uitgebreidheid en Fast in de regel, plus startprompts, Talk Mode, spraaknotities en Luisteren. De redenering en toolactiviteit van de assistent blijven verborgen in deze compacte weergave.

## Meerdere Gateway-vensters

Open **Settings → Gateways** om herbruikbare Gateway-profielen toe te voegen of te verwijderen. Elk
profiel bevat een privénetwerk-`ws://`- of beveiligd `wss://`-eindpunt en het
optionele token of wachtwoord; referenties worden opgeslagen in de macOS-sleutelhanger.
Beveiligde profielen onderhouden hun eigen, door systeemvertrouwen beveiligde certificaatpin voor het eerste gebruik
en nemen `gateway.remote.tlsFingerprint` niet over van de primaire Gateway.
Als een profiel wordt verwijderd, worden ook de geopende vensters ervan gesloten en wordt de secundaire
verbinding beëindigd.

Kies **File → New Gateway Window…** of druk op Cmd-N en selecteer vervolgens een van die
opgeslagen profielen. De kiezer onthoudt het laatst gebruikte profiel. Elke
selectie maakt een nieuw onafhankelijk venster aan, zodat dezelfde Gateway in
meerdere vensters kan worden weergegeven met verschillende actieve sessies en navigatiestatussen.

Elk opgeslagen profiel beheert één gedeelde Gateway-verbinding, apparaatverificatiebereik,
transcriptcache, offline uitgaande wachtrij en routeleases. Vensters voor dat profiel
hergebruiken deze middelen, maar kunnen onafhankelijk worden genavigeerd. Vensters voor
verschillende profielen blijven verbonden en voeren tegelijkertijd chats uit.

De geconfigureerde Gateway van de menubalkapp blijft de eigenaar van de
mogelijkheden van de Mac-node en Talk Mode. Aanvullende Gateway-vensters zijn uitsluitend voor operators, zodat een
tweede Gateway de algemene microfoon- of apparaatbediening niet ongemerkt op een ander doel kan richten.
Luisteren/TTS en normale chatacties gebruiken de eigen Gateway-verbinding van het venster.

## Quick Chat-balk

Druk op Option-Space (⌥Space) of kies **Quick Chat** in het menubalkmenu om een zwevende opsteller voor de hoofdsessie te openen. Wijzig de algemene sneltoets met de recorder in **Settings → General → Quick Chat shortcut**.

Quick Chat toont de doelagent (avatar of emoji, met de naam van de agent als tijdelijke aanduiding) en verzendt naar de hoofdsessie van die agent. Nadat Return een verzending accepteert, blijft de balk geopend en wordt deze naar beneden uitgebreid met het gestreamde Markdown-antwoord en het recente transcript. Het invoerveld van de balk blijft de opsteller. Druk op Command-Return om te verzenden en hetzelfde doel in het volledige chatvenster te openen, op Shift-Return voor een nieuwe regel of op Escape om de volledige balk en het antwoordgebied te sluiten. Ook door erbuiten te klikken wordt deze gesloten. Wanneer relevante macOS-machtigingen ontbreken, biedt een gekoppelde strook de acties **Toestaan** en **Niet nu**.

Gebruik de microfoonknop om tekst in de opsteller te dicteren. Gedeeltelijke spraakresultaten vervangen het gedicteerde gedeelte live, terwijl tekst die al in de opsteller stond behouden blijft. Druk nogmaals op de knop, op Return of op Escape om te stoppen; bij verzenden, verbergen of het verliezen van de focus van Quick Chat wordt de microfoon eveneens vrijgegeven. Bij het eerste gebruik wordt om toegang tot de macOS-microfoon en spraakherkenning gevraagd. Quick Chat gebruikt Apple Speech en kan de netwerkdiensten daarvan gebruiken; alleen passieve Voice Wake vereist herkenning op het apparaat.

Het compacte modelbedieningselement toont het huidige model en redeneerniveau van de doelsessie. Een modelkeuze werkt die sessie bij en blijft daar dus behouden, terwijl een redeneerkeuze alleen geldt voor elk bericht dat vanuit de huidige Quick Chat-weergave wordt verzonden. Lokale keuzes worden opnieuw ingesteld wanneer de balk wordt verborgen. Bij het wisselen van agent of het kiezen van een recente sessie blijven expliciete keuzes behouden, maar wordt de onderliggende modelstatus van de nieuw gekozen doelsessie opnieuw geladen.

Klik op de geschiedenisknop om uit de vijf laatst bijgewerkte sessies te kiezen of terug te keren naar **Nieuw bericht aan &lt;agent&gt;**. Een recente selectie verzendt naar exact die sessie en wijzigt de tijdelijke aanduiding in **Antwoorden in &lt;session&gt;**. Door Quick Chat te verbergen, wordt dit tijdelijke doel opnieuw ingesteld op de hoofdsessie van de geselecteerde agent; door via het avatarmenu van agent te wisselen, wordt het ook gewist.

Command-Return opent het gesprek van de agent die de verzending heeft ontvangen, ook wanneer het sessiebereik algemeen is.

De cameraknop opent een menu voor **Venster vastleggen…** of **Gebied vastleggen…**. Bij het vastleggen van een venster krijgt elk zichtbaar venster een label; bij het vastleggen van een gebied wordt elk scherm gedimd terwijl je een gebied sleept en wordt de actuele grootte ervan getoond. De geselecteerde schermafbeelding wordt naar de gekozen agent verzonden, met eventuele getypte tekst als bijschrift. Bij het eerste gebruik wordt om toegang tot macOS-schermopname gevraagd. Escape, klikken op lege ruimte of klikken zonder een betekenisvol gebied te slepen annuleert de actie.

Gebruik de knop met documenttekst om tekst uit het venster met focus van de app met focus bij te voegen. Quick Chat toont het resultaat als een verwijderbare contextchip in plaats van de vastgelegde tekst in de opsteller te plaatsen; bij verzending wordt de tekst van de chip aan het uitgaande bericht toegevoegd en daarna gewist. Hiervoor is de macOS-machtiging voor toegankelijkheid vereist. Bijgevoegde tekst wordt ook gewist wanneer Quick Chat wordt gesloten, zodat context van de ene weergave niet in een latere verzending kan terechtkomen.

Kies nadat een antwoord is voltooid **Plakken in &lt;app&gt;** om de zichtbare assistenttekst ervan, zonder verborgen redenering, naar het algemene klembord te kopiëren en in de app te plakken die op de voorgrond stond. Hiervoor is de macOS-machtiging voor toegankelijkheid vereist. De actie vervangt de huidige klembordinhoud en verbergt vervolgens Quick Chat.

Schakel de functie volledig uit via **Settings → General → Quick Chat**; dezelfde sectie bevat de sneltoetsrecorder.

- **Lokale modus**: maakt rechtstreeks verbinding met de lokale Gateway-WebSocket.
- **Externe modus**: gebruikt de geconfigureerde rechtstreekse `ws://`- of `wss://`-route of de door de app beheerde SSH-tunnel als datavlak.

## Starten en fouten opsporen

- Handmatig: Lobster-menu -> "Chat openen".
- Automatisch openen voor tests:

  ```bash
  dist/OpenClaw.app/Contents/MacOS/OpenClaw --chat
  ```

  (`--webchat` wordt geaccepteerd als verouderde alias.)

- Logboeken: `./scripts/clawlog.sh` (subsysteem `ai.openclaw`, categorie `WebChatSwiftUI`).

## Hoe het is aangesloten

- Datavlak: Gateway-WS-methoden `chat.history`, `chat.message.get`, `chat.send`, `chat.abort`, `chat.inject`, plus `question.list` en `question.resolve`, en gebeurtenissen `chat`, `agent`, `presence`, `tick`, `health`; vraagkaarten volgen gebeurtenissen `question.requested` en `question.resolved` en worden na nieuwe verbindingen vernieuwd vanuit `question.list`.
- `chat.history` retourneert een voor weergave genormaliseerd transcript: inline directivetags worden uit zichtbare tekst verwijderd, XML-payloads voor toolaanroepen in platte tekst (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`, inclusief afgekorte blokken) en uitgelekte tokens voor modelbesturing worden verwijderd, assistentrijen die uitsluitend uit stille tokens bestaan, zoals exact `NO_REPLY`/`no_reply`, worden weggelaten en te grote rijen kunnen worden vervangen door een afgekorte tijdelijke aanduiding.
- Sessie: gebruikt standaard de primaire sessie zoals hierboven; via de UI kan tussen sessies worden gewisseld.
- Sessiegroepen: `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` en `sessions.groups.delete` beheren de groepscatalogus. Het lidmaatschap is de sessie-`category`, bijgewerkt via `sessions.patch`.
- Ongelezen status: nadat een sessie is geactiveerd en de livegeschiedenis ervan met succes is geladen, wist de app de ongelezen markering van die sessie. Mislukte laadpogingen van de geschiedenis wissen deze niet; een tijdelijke fout bij het toepassen van een patch wordt bij de volgende activering opnieuw geprobeerd.
- Onboarding gebruikt een afzonderlijke sessie om de configuratie bij het eerste gebruik gescheiden te houden.
- Offlinecache: de app bewaart per Gateway een kleine alleen-lezen cache met recente chatsessies en transcripten (`~/Library/Application Support/OpenClaw/chat-cache.sqlite`): bij een koude start wordt het laatst bekende transcript onmiddellijk weergegeven en vernieuwd zodra de Gateway reageert, en recente chats blijven tijdens een verbroken verbinding doorzoekbaar (verzenden blijft uitgeschakeld totdat de verbinding is hersteld).

## Beveiligingsoppervlak

- De externe modus stuurt alleen de Gateway-WebSocket-besturingspoort door via SSH.

## Bekende beperkingen

- De UI is geoptimaliseerd voor chatsessies, niet als volledige browsersandbox.

## Gerelateerd

- [WebChat](/nl/web/webchat)
- [macOS-app](/nl/platforms/macos)
