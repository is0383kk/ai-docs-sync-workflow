---
read_when:
    - Je wilt dat een agent een interactief resultaat weergeeft in webchat, een native app of Discord
    - Je wilt dat widgetknoppen vervolgprompts naar de chat sturen
    - Je wilt widgets vormgeven met de gedeelde ontwerptokens
    - Je hebt het contract voor invoer, beveiliging of retentie van show_widget nodig
sidebarTitle: Show widget
summary: Zelfstandige HTML-widgets weergeven in ondersteunde chatomgevingen
title: Widget tonen
x-i18n:
    generated_at: "2026-07-27T06:36:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 903adff1fadeb9d224d3e2d839c86082b5244e1e319255c8d3f6619344b749a3
    source_path: tools/show-widget.md
    workflow: 16
---

`show_widget` is een kerntool die een zelfstandige HTML-widget weergeeft op het huidige oppervlak van de gebruiker. OpenClaw geeft deze inline weer in de Control UI en in transcripties van Quick Chat op iOS, Android, macOS en Linux; het Linux-dashboard gebruikt de Control UI van de browser. In een Discord-sessie waarin [Activities](/channels/discord-activities) is ingeschakeld, plaatst de Discord-plugin een knop **Widget openen** waarmee deze als Activity wordt gestart.

## Hoe widgets werken

Wanneer de agent `show_widget` aanroept, verpakt de OpenClaw-kern `widget_code` in een minimaal HTML-document, slaat dit op als een Canvas-document en retourneert een preview-handle. De Control UI geeft die handle weer in een gesandboxte iframe, terwijl Quick Chat op iOS, Android, macOS en Linux geïsoleerde webweergaven gebruikt. Volledige chatclients herstellen de widget nadat de geschiedenis opnieuw is geladen; Quick Chat behoudt de widget voor het actieve antwoord.

In Control UI-sessies kan een Canvas-widget ook aan het sessiedashboard worden vastgemaakt. Stel `pin: true` in bij de toolaanroep of gebruik **Vastmaken aan dashboard** op een bestaande widget in de transcriptie. Vastgemaakte HTML draait achter dezelfde sandboxhost met een toegewezen origin en dubbele iframe die door MCP Apps wordt gebruikt; de browser verwerkt nooit een gegevensbinding van een widget binnen het niet-vertrouwde frame.

Voor insluiting in de browser injecteert het wrapperdocument vier kleine hostbridges rond de widgetcode:

- Een grootterapporteur stuurt de hoogte van de weergegeven inhoud naar de insluitende chat, die deze begrenst en de iframe passend maakt (160 tot 1200 pixels).
- Een hostbridge definieert de verouderde helper `sendPrompt(text)` plus de gestructureerde API's `openclaw.prompt`, `openclaw.state`, `openclaw.data` en `openclaw.cron`. Inline chatprompts behouden hun privéberichtkanaal; dashboard-API's gebruiken een aanvraagkanaal dat aan een weergaveticket is gebonden. Zie [Interactieve widgets](#interactive-widgets) en [Dashboardmogelijkheden](#dashboard-capabilities).
- Een themabridge luistert naar de huidige ontwerptokens van de Control UI en past deze toe als CSS-variabelen, bij het laden en opnieuw bij elke themawijziging.
- Een snapshotbridge geeft het huidige widgetdocument weer als PNG wanneer de insluitende chat om export vraagt.

Al het overige blijft binnen het frame: het document draait in een ondoorzichtige origin met een strikt Content Security Policy, zodat widgetscripts geen toegang hebben tot de Control UI, de Gateway of het netwerk.

De kernimplementatie is alleen beschikbaar wanneer de oorspronkelijke Gateway-client de mogelijkheid `inline-widgets` declareert. De Control UI en ondersteunde native apps declareren deze mogelijkheid automatisch. Quick Chat op Linux blijft uitsluitend tekst weergeven voor Gateway-verbindingen die een aangepaste TLS-leafpin vereisen, omdat de WebView van het platform die pin niet kan binden. De Discord-implementatie is alleen beschikbaar in Discord-sessies waarvoor Activities zijn geconfigureerd. Andere kanaaluitvoeringen ontvangen `show_widget` niet.

Mogelijkheidstransport ondersteunt ingesloten modelbackends, modelbackends via de Codex-appserver en modelbackends via de CLI. Met grants geauthenticeerde MCP-aanroepers en rechtstreekse HTTP-toolaanroepers blijven standaard geblokkeerd omdat ze geen clientmogelijkheden declareren.

## Ontwerpsysteem

Elke Canvas-widget bevat een basisstylesheet zonder klassen en een kleine set tokens:

| Token                                                                                 | Doel                                  |
| ------------------------------------------------------------------------------------- | ------------------------------------- |
| `--surface`                                                                           | Oppervlaktekleur op paginaniveau      |
| `--card`                                                                              | Achtergrond van kaarten, knoppen en code |
| `--elevated`                                                                          | Verhoogde achtergrond van formulierbesturingselementen |
| `--text`                                                                              | Standaardtekst van hoofdinhoud en besturingselementen |
| `--text-strong`                                                                       | Koppen en prominente waarden          |
| `--muted`                                                                             | Secundaire tekst en subtiele randen   |
| `--border`                                                                            | Standaardscheidingen en kaartranden   |
| `--border-strong`                                                                     | Sterke randen van besturingselementen |
| `--accent`                                                                            | Links en focusringen                  |
| `--accent-fill`                                                                       | Vulling van de primaire actie         |
| `--accent-fg`                                                                         | Tekst op een primaire actie           |
| `--ok`                                                                                | Successtatus                          |
| `--warn`                                                                              | Waarschuwingsstatus                   |
| `--danger`                                                                            | Foutstatus of destructieve status     |
| `--info`                                                                              | Informatieve status                   |
| `--radius`                                                                            | Gedeelde hoekradius voor besturingselementen en kaarten |
| `--font-body`                                                                         | Lettertypestack voor hoofdtekst van de host |
| `--font-mono`                                                                         | Monospace-lettertypestack van de host |
| `--accent-subtle`, `--ok-subtle`, `--warn-subtle`, `--danger-subtle`, `--info-subtle` | Afgeleide doorschijnende statusachtergronden |

Koppen, alinea's, links, knoppen, invoervelden, keuzelijsten, tekstgebieden, tabellen en codeblokken zonder klassen krijgen basisstijlen. Helperklassen bieden veelvoorkomende patronen:

- `.card` voor een inhoudsoppervlak met rand
- `.badge`, plus `.ok`, `.warn`, `.danger` of `.info`, voor compacte statuslabels
- `.metric` voor een prominente numerieke waarde
- `.muted` voor secundaire tekst
- `.row` voor een horizontale indeling met regelomloop
- `button.primary` voor de primaire actie

De Control UI stuurt een `openclaw:widget-theme`-bericht met de actieve themawaarden wanneer een widget wordt geladen en telkens wanneer het thema verandert. Widgets volgen daardoor elke themafamilie, waaronder Claw, Knot, Dash en aangepaste thema's, zonder opnieuw te laden. Buiten de Control UI, waaronder native apps en rechtstreeks geopende widgets, gebruiken widgets het ingebouwde lichte of donkere palet dat door `prefers-color-scheme` is geselecteerd.

Ontwerp widgets volgens drie regels:

1. Gebruik de ontwerpvariabelen voor elke kleur en achtergrond. Leg geen kleurwaarden hard vast.
2. Houd de pagina-achtergrond transparant, zodat de widget bij het hostoppervlak past.
3. Reserveer `--accent-fill` voor maximaal één primaire actie.

**Exporteren:** Open in webchat het menu van de widgetkaart om de weergegeven widget naar het klembord te kopiëren of als PNG te downloaden. Bij oudere widgetdocumenten zonder snapshotbridge wordt teruggevallen op het downloaden van een HTML-bestand.

## De tool gebruiken

Beide implementaties gebruiken dezelfde verplichte velden:

<ParamField path="title" type="string" required>
  Korte titel die bij de inline preview en als titel van het gehoste document wordt weergegeven.
</ParamField>

<ParamField path="widget_code" type="string" required>
  Zelfstandige HTML of SVG. Voor clients met inline widgets wordt invoer die na het verwijderen van witruimte begint met `<svg` weergegeven in SVG-modus; de maximale lengte is 262.144 tekens. Discord accepteert een volledig HTML-document of hoofdtekstfragment van maximaal 48 KiB.
</ParamField>

Discord accepteert ook optionele `button_label`-tekst voor de startknop van de Activity. Het Canvas-schema laat dit veld dat uitsluitend voor Discord bestemd is bewust weg.

De Canvas-kerntool accepteert deze optionele velden voor plaatsing op het dashboard:

- `pin`: plaats de widget ook op het sessiedashboard.
- `name`: stabiele widgetnaam; gebruikt standaard een slug van `title`.
- `tab`: slug van het doeltabblad.
- `size`: een van `sm`, `md`, `lg`, `xl` of `full`.
- `after`: naam van de widget waarna deze widget moet worden geplaatst.
- `capabilities`: toegang die door een vastgemaakte widget wordt aangevraagd. `netOrigins` bevat exacte HTTPS-origins; `tools` bevat `prompt`, een toegestane leesbinding of een exacte `cron.trigger:<jobId>`-actie.

Het kernresultaat bevat een Canvas-preview-handle, zodat de Control UI en ondersteunde native apps de widget rechtstreeks vanuit de toolaanroep weergeven en deze herstellen nadat de geschiedenis opnieuw is geladen. Vastgemaakte resultaten behouden ook de widgetnaam op het bord, zodat de Control UI na het opnieuw laden van de transcriptie geen dubbele mogelijkheid tot vastmaken aanbiedt. Discord retourneert de opgeslagen widget- en bericht-ID's.

`discord_widget` blijft gedurende één release geregistreerd als verouderde alias. Nieuwe agentaanroepen moeten `show_widget` gebruiken.

## Interactieve widgets

In de Control UI kunnen widgetscripts het gesprek aansturen. Het wrapperdocument definieert een globale functie `sendPrompt(text)`; wanneer deze wordt aangeroepen, wordt `text` naar de chat verzonden alsof de gebruiker het bericht had getypt en verzonden. Koppel deze aan knoppen of andere besturingselementen om interactieve flows te bouwen, zoals keuzelijsten, quizzen of dashboards met detailnavigatie. Native apps geven interactieve widgetcode weer, maar stellen deze chatpromptbridge niet beschikbaar.

```html
<button onclick="sendPrompt('Toon de mislukte tests in detail')">Mislukte tests</button>
```

Elke prompt wordt aan beide zijden van de framegrens gevalideerd:

- `sendPrompt` vereist [tijdelijke gebruikersactivering](https://developer.mozilla.org/en-US/docs/Web/Security/User_activation) binnen de widget: dit werkt alleen gedurende de paar seconden nadat de gebruiker in de widget klikt of een toets indrukt. Koppel het daarom aan knoppen en andere klikdoelen — automatisch aanroepen tijdens het laden doet niets. De bridge houdt het verzendende eindpunt voor zichzelf privé en blokkeert standaard in browsers die geen gebruikersactivering beschikbaar stellen, zodat widgetcode de controle niet kan omzeilen.
- De bevoegdheid voor prompts behoort alleen toe aan het oorspronkelijke widgetdocument. De vertrouwde bridge biedt zijn kanaaleindpunt aan de chat aan voordat widgetcode kan worden uitgevoerd of door het frame kan navigeren, de chat accepteert alleen dat eerste aanbod en het kanaal eindigt samen met het document bij navigatie. Extern toegestane insluit-URL's worden nooit geaccepteerd.
- Het widgetframe moet zichtbaar zijn in de chattranscriptie en focus hebben — een aanvullend door de host waargenomen signaal dat de gebruiker daadwerkelijk met deze widget communiceert.
- De tekst mag na het verwijderen van witruimte niet leeg zijn en mag maximaal 4.000 tekens bevatten.
- Prompts die beginnen met `/` worden geweigerd, zodat widgetcode geen chatopdrachten zoals `/approve` of `/stop` kan activeren.
- Elk widgetdocument mag maximaal 10 prompts per voortschrijdende minuut verzenden; overtollige prompts worden stilzwijgend verwijderd.

Geaccepteerde prompts verschijnen in de transcriptie als gewone gebruikersberichten en starten een normale agentbeurt in de sessie waartoe de widget behoort. Er is geen feedbackkanaal naar de widget: een verwijderde prompt mislukt stilzwijgend en de widget kan het antwoord van de agent niet lezen.

## Dashboardmogelijkheden

Vastgemaakte widgets kunnen één ticketgebonden host-API gebruiken nadat de operator de declaratie op de kaart in afwachting heeft beoordeeld:

- `openclaw.prompt.send(text)` vereist tijdelijke gebruikersactivering en plaatst een zichtbaar bericht in het invoerveld. Door de `prompt`-tooltoekenning te declareren en te ontvangen, wordt de extra bevestiging per klik overgeslagen; validatie, focuscontroles en frequentielimieten blijven van toepassing.
- `openclaw.state.emit(payload)` voegt een sessiemelding toe. Payloads zijn beperkt tot 8 KiB en identieke clientemissies binnen vijf seconden worden samengevoegd.
- `openclaw.data.read(bindingId, params?)` wordt alleen bij de Gateway verwerkt. Toekenbare bindingen zijn `sessions.list`, `usage.status`, `usage.cost`, `cron.list`, `cron.status`, `agents.list` en `health`.
- `openclaw.cron.trigger(jobId)` voert een bestaande taak alleen nu uit wanneer de exacte `cron.trigger:<jobId>`-mogelijkheid is toegekend.

Netwerktoegang staat los van hosttools. Plaats exacte HTTPS-origins in `capabilities.netOrigins`; na goedkeuring worden alleen die origins opgenomen in de `connect-src` van de widget. Jokertekens, aanmeldgegevens, paden, querystrings en niet-gedeclareerde origins blijven geblokkeerd. Een letterlijke poort is alleen toegestaan wanneer deze deel uitmaakt van de gedeclareerde origin.

## Beveiliging en opslag

Widgetdocumenten gebruiken beperkende Content Security Policies. Inline stijlen en scripts zijn toegestaan, terwijl het laden van externe resources geblokkeerd blijft. Inline transcriptwidgets hebben geen toegang tot het netwerk. Een vastgezette dashboardwidget kan alleen gegevens ophalen van exacte HTTPS-origins die de agent heeft gedeclareerd en de beheerder heeft toegekend.

In het iframe van de Control UI wordt `allow-same-origin` altijd weggelaten, zelfs wanneer de algemene insluitmodus `trusted` is, zodat widgetscripts de origin van de bovenliggende applicatie niet kunnen lezen. Native clients gebruiken geïsoleerde, niet-persistente webviews en blokkeren navigatie weg van de gehoste widget. De host voor kerndocumenten levert widgets ook met een `Content-Security-Policy: sandbox allow-scripts`-responseheader, zodat bij directe weergave de widget nog steeds in een ondoorzichtige origin wordt uitgevoerd in plaats van in een applicatie-origin. Render alleen widgetcode die je in dat geïsoleerde frame wilt uitvoeren.

Het iframe volgt ook [`gateway.controlUi.embedSandbox`](/nl/web/control-ui#hosted-embeds). Het standaardniveau `scripts` ondersteunt interactieve widgets en behoudt daarbij origin-isolatie.

Het geaccepteerde WebRTC-datachannelrestant voor uitgaand verkeer wordt beschreven in [Dashboardarchitectuur](/web/dashboard-architecture#modeled-residual-webrtc-data-channels).

Canvas bewaart maximaal 32 widgets per sessie (of per agent wanneer er geen sessie beschikbaar is). Als je nog een widget maakt, wordt het oudste document binnen dat bereik verwijderd.

## Gerelateerd

- [Gehoste insluitingen van de Control UI](/nl/web/control-ui#hosted-embeds)
- [Discord Activities](/channels/discord-activities)
- [Canvas-nodebesturing](/nl/plugins/reference/canvas)
- [Clientmogelijkheden van het Gateway-protocol](/nl/gateway/protocol#client-capabilities)
