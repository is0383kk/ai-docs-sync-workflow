---
read_when:
    - WebChat-toegang debuggen of configureren
summary: Statische loopbackhost voor WebChat en gebruik van Gateway-WS voor de chatinterface
title: WebChat
x-i18n:
    generated_at: "2026-07-27T06:18:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19c301af1eb1b28650849cdd90924805dd0f5189516693505d9b75f62197007f
    source_path: web/webchat.md
    workflow: 16
---

Status: de macOS/iOS SwiftUI-chatinterface communiceert rechtstreeks met de Gateway-WebSocket. Geen ingebouwde browser, geen lokale statische server.

## Wat het is

- Een native chatinterface voor de Gateway.
- Gebruikt dezelfde sessies en routeringsregels als andere kanalen.
- Deterministische routering: antwoorden gaan altijd terug naar WebChat.
- De geschiedenis wordt altijd opgehaald van de Gateway (lokale bestanden worden niet gevolgd). Als de Gateway onbereikbaar is, is WebChat alleen-lezen.

## Snel aan de slag

1. Start de Gateway.
2. Open de WebChat-interface (macOS/iOS-app) of het chattabblad van de Control UI.
3. Zorg dat een geldig authenticatiepad voor de Gateway is geconfigureerd (standaard een gedeeld geheim, zelfs op loopback).

## Hoe het werkt

- De interface maakt verbinding met de Gateway-WebSocket en gebruikt de RPC-methoden `chat.history`, `chat.send`, `chat.inject` en `chat.message.get`.
- `chat.history` is begrensd voor stabiliteit: de Gateway kan lange tekstvelden afkappen, omvangrijke metagegevens weglaten en te grote items vervangen door `[chat.history omitted: message too large]`. API-clients kunnen per aanvraag een `maxChars` verzenden om de standaardlimiet voor één aanroep te overschrijven.
- Wanneer een zichtbaar assistentbericht in `chat.history` is afgekapt, kan de Control UI een zijpaneel openen en het volledige, voor weergave genormaliseerde item op aanvraag ophalen via `chat.message.get`, zonder de standaardpayload van de geschiedenis te vergroten. `chat.message.get` gebruikt dezelfde transcriptvertakking en weergaveregels als `chat.history`, maar richt zich op één item via `messageId` en geeft een waarheidsgetrouwe reden voor onbeschikbaarheid terug wanneer de volledige inhoud niet meer kan worden geretourneerd.
- `chat.history` volgt de actieve transcriptvertakking voor sessiebestanden waaraan alleen wordt toegevoegd, zodat verlaten herschrijfvertakkingen en vervangen kopieën van prompts niet in WebChat worden weergegeven.
- Compaction-items worden weergegeven als een scheidingslijn met de tekst "Gecompacteerde geschiedenis", die uitlegt dat het gecompacteerde transcript als controlepunt wordt bewaard, met een actie om sessiecontrolepunten te openen (vertakken of herstellen, wanneer de machtigingen dit toestaan).
- De Control UI onthoudt de onderliggende Gateway-`sessionId` die door `chat.history` wordt geretourneerd en neemt deze op in volgende `chat.send`-aanroepen, zodat na opnieuw verbinden en het vernieuwen van de pagina hetzelfde opgeslagen gesprek wordt voortgezet, tenzij de gebruiker een sessie start of opnieuw instelt.
- Verzendingen op de voorgrond bevatten ook het blad van de weergegeven vertakking uit de weergegeven geschiedenis als `expectedLeafEntryId`; als een andere client eerst van vertakking is gewisseld, zet de Control UI het bericht klaar voor controle en vernieuwt deze het transcript in plaats van het bericht in de nieuwe vertakking te plaatsen. Bij opnieuw verbinden en het opnieuw afspelen van een herstelde outbox wordt deze voorwaarde na afstemming met de huidige geschiedenis bewust weggelaten.
- `chat.send` accepteert een idempotentiesleutel (de Control UI gebruikt de uitvoerings-id); de Gateway dedupliceert herhaalde aanvragen die dezelfde sleutel hergebruiken, zodat opnieuw uitgevoerde of dubbele lopende verzendingen voor dezelfde sessie/hetzelfde bericht/dezelfde bijlagen geen tweede uitvoering maken.
- Bij het beantwoorden van een specifiek bericht (rechtsklikken → Reply) wordt de transcript-id van het doel als `replyToId` verzonden bij `chat.send`. De Gateway haalt dat bericht op uit de sessiegeschiedenis en vult dezelfde kanaalonafhankelijke metagegevens voor antwoordcontext in die Discord-antwoorden gebruiken: agents zien `has_reply_context` plus het niet-vertrouwde blok "Antwoorddoel van huidig gebruikersbericht" met het afzenderlabel en de berichttekst. (Webchat-prompts blijven vluchtige gespreks-id's zoals `reply_to_id` onderdrukken, overeenkomstig het bestaande byte-stabiele promptbeleid voor rechtstreekse webchatsessies.) Antwoorddoelen zonder persistente transcript-id (bijvoorbeeld wachtende verzendingen) vallen terug op een inline citaat in de berichttekst.
- Opstartbestanden van de werkruimte en wachtende `BOOTSTRAP.md`-instructies worden aangeleverd via de sectie `# Project Context` van de systeemprompt van de agent en niet naar het WebChat-gebruikersbericht gekopieerd. Als opstartinhoud wordt afgekapt, krijgt de systeemprompt in plaats daarvan een korte "Melding over opstartcontext"; gedetailleerde aantallen en configuratie-instellingen blijven beschikbaar op diagnostische oppervlakken.
- Weergavenormalisatie op `chat.history` verwijdert: OpenClaw-context die alleen tijdens runtime wordt gebruikt, wrappers voor inkomende enveloppen, inline tags met afleveringsrichtlijnen zoals `[[reply_to_current]]`, `[[reply_to:<id>]]` en `[[audio_as_voice]]`, XML-payloads in platte tekst voor toolaanroepen (`<tool_call>`, `<function_call>`, `<tool_calls>`, `<function_calls>`, inclusief afgekapt blokken) en gelekte ASCII-/volledige-breedtebesturingstokens van modellen. Assistentitems waarvan de volledige zichtbare tekst alleen het stille token `NO_REPLY` is (hoofdletterongevoelig), worden weggelaten.
- Antwoordpayloads die als redenering zijn gemarkeerd (`isReasoning: true`), worden uitgesloten van WebChat-assistentinhoud, tekst bij het opnieuw afspelen van het transcript en blokken met audio-inhoud, zodat payloads die alleen denkstappen bevatten niet verschijnen als zichtbare assistentberichten of afspeelbare audio.
- `chat.inject` voegt rechtstreeks een assistentnotitie toe aan het transcript en zendt deze uit naar de interface (geen agentuitvoering).
- Bij afgebroken uitvoeringen kan gedeeltelijke assistentuitvoer zichtbaar blijven in de interface. De Gateway slaat die gedeeltelijke tekst op in de transcriptgeschiedenis wanneer gebufferde uitvoer bestaat en markeert het item met metagegevens over de afbreking.

### Transcript- en afleveringsmodel

WebChat heeft twee afzonderlijke gegevenspaden:

- De SQLite-transcriptrijen vormen het duurzame model-/runtimetranscript. Voor normale agentuitvoeringen slaat de ingebouwde OpenClaw-runtime modelzichtbare `user`-, `assistant`- en `toolResult`-berichten op via de sessietoegang. WebChat schrijft geen willekeurige afleverings-, status- of hulptekst naar dat transcript.
- Gateway-`ReplyPayload`-gebeurtenissen zijn de live afleveringsprojectie: genormaliseerd voor weergave in WebChat/kanalen, blokstreaming, richtlijntags, media-insluiting, TTS-/audiovlaggen en terugvalgedrag van de interface. Ze vormen zelf niet het canonieke sessielogboek.
- Testomgevingen die zichtbare antwoorden via `tools.message` vereisen, gebruiken WebChat nog steeds als interne bronbestemming voor antwoorden tijdens de huidige uitvoering. Een doelloze `message.send` van die actieve WebChat-uitvoering wordt in dezelfde chat geprojecteerd en naar het sessietranscript gespiegeld; WebChat wordt geen herbruikbaar uitgaand kanaal en neemt nooit `lastChannel` over.
- WebChat voegt alleen assistentitems aan het transcript toe wanneer de Gateway eigenaar is van een weergegeven bericht buiten een normale ingebouwde agentbeurt: `chat.inject`, antwoorden op opdrachten zonder agent, afgebroken gedeeltelijke uitvoer en door WebChat beheerde media-aanvullingen voor het transcript.
- Als tijdens een uitvoering live assistenttekst verschijnt maar na het opnieuw laden van de geschiedenis verdwijnt, controleer dan in deze volgorde: of het SQLite-transcript de assistenttekst bevat, of de weergaveprojectie van `chat.history` deze heeft verwijderd en vervolgens of de samenvoeging van de optimistische staart in de Control UI de lokale afleveringsstatus heeft vervangen door de persistente momentopname.

Definitieve antwoorden van normale agentuitvoeringen moeten duurzaam zijn omdat de ingebouwde runtime de assistent-`message_end` schrijft. Elke terugvaloptie die een afgeleverde definitieve payload naar het transcript spiegelt, moet eerst voorkomen dat een assistentbeurt wordt gedupliceerd die al door de ingebouwde runtime is geschreven.

## Toolpaneel voor agents in de Control UI

- Het toolpaneel `/agents` van de Control UI heeft een weergave "Nu beschikbaar" die wordt aangestuurd door `tools.effective(sessionKey=...)`: een door de server afgeleide, alleen-lezen projectie van de toolinventaris van de huidige sessie, inclusief kern-, Plugin-, kanaaleigen tools en reeds ontdekte tools van MCP-servers.
- Een afzonderlijke weergave voor configuratiebewerking (aangestuurd door `tools.catalog`) omvat profielen, overschrijvingen per agent en catalogussemantiek.
- Beschikbaarheid tijdens runtime is sessiegebonden. Wisselen tussen sessies van dezelfde agent kan de lijst "Nu beschikbaar" wijzigen. Als geconfigureerde MCP-servers sinds de laatste ontdekking niet zijn verbonden of gewijzigd, toont het paneel een melding in plaats van stilzwijgend MCP-transporten te starten vanuit het leespad.
- De configuratie-editor impliceert geen beschikbaarheid tijdens runtime; daadwerkelijke toegang volgt nog steeds de beleidsprioriteit (`allow`/`deny`, overschrijvingen per agent en per provider/kanaal).

## Gebruik op afstand

- De externe modus tunnelt de Gateway-WebSocket via SSH/Tailscale.
- Je hoeft geen afzonderlijke WebChat-server uit te voeren.

## Configuratiereferentie (WebChat)

Volledige configuratie: [Configuratie](/nl/gateway/configuration)

WebChat heeft geen persistente configuratiesectie. De Gateway gebruikt de ingebouwde weergavelimiet `chat.history`; API-clients kunnen per aanvraag `maxChars` verzenden om deze voor één aanroep te overschrijven. Verouderde configuratie voor `channels.webchat` en `gateway.webchat` is buiten gebruik gesteld; voer `openclaw doctor --fix` uit om deze te verwijderen.

Gerelateerde algemene opties:

- `gateway.port`, `gateway.bind`: WebSocket-host/-poort.
- `gateway.auth.mode`, `gateway.auth.token`, `gateway.auth.password`:
  WebSocket-authenticatie met een gedeeld geheim.
- `gateway.auth.allowTailscale`: het chattabblad van de Control UI in de browser kan Tailscale
  Serve-identiteitsheaders gebruiken wanneer dit is ingeschakeld.
- `gateway.auth.mode: "trusted-proxy"`: reverse-proxy-authenticatie voor browserclients achter een identiteitsbewuste **niet-loopback**-proxybron (zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth)).
- `gateway.remote.url`, `gateway.remote.token`, `gateway.remote.password`: extern Gateway-doel.
- `session.*`: sessieopslag en standaardwaarden voor de hoofdsleutel.

## Gerelateerd

- [Control UI](/nl/web/control-ui)
- [Dashboard](/nl/web/dashboard)
