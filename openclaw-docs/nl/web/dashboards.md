---
read_when:
    - Sessiedashboards in de Control UI gebruiken of uitleggen
    - Bepalen wat agents op een bord kunnen doen en waarvoor toestemming van een operator nodig is
summary: 'Sessiedashboards: door agents gebouwde widgets, borden, tabbladen en de vastgezette chat'
title: Sessiedashboards
x-i18n:
    generated_at: "2026-07-27T05:21:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3babbc859e261aa959740ea778b44fdc1a07bce8ce7628cbabcfbc5fa207a0ce
    source_path: web/dashboards.md
    workflow: 16
---

Elke thread in de Control UI heeft twee gezichten: het gesprek dat je kent en een
**dashboard** — een raster met live widgets die je agent voor je bouwt. Een thread
zonder widgets is gewoon een chat. Zodra een widget wordt vastgemaakt, verschijnt
een schakelaar **Chat | Dashboard** in de koptekst en wordt het dashboard het
hoofdscherm, met je chat ernaast vastgezet.

Je hoeft niets in te stellen en er is geen aparte app om te configureren: dashboards
zijn een kernfunctie, behoren tot de thread, worden bij de agent opgeslagen en blijven
behouden na `/new` en `/reset` (de gesprekscontext wordt gewist; het bord blijft behouden).

## Bouw een dashboard door erom te vragen

Vraag je agent wat je wilt zien:

> Maak een widget met de naam revenue-graph: een interactief staafdiagram van de
> maandelijkse omzet. Voeg de knoppen "Bars" en "Trend" toe om tussen weergaven
> te schakelen. Maak deze vast aan mijn dashboard.

De agent geeft de widget eerst inline in de chat weer, zodat je deze kunt bekijken
voordat deze ergens wordt geplaatst. Vervolgens:

- **Je maakt deze vast**: beweeg de muis over een inline widget en kies **Vastmaken aan dashboard**.
- **Of de agent maakt deze rechtstreeks vast** wanneer je daarom vraagt en werkt deze later bij
  op naam — widgets hebben vaste namen, dus "werk revenue-graph bij met de
  cijfers van juni" vervangt de inhoud ter plekke, terwijl het bord blijft staan.

Widgets zijn kleine, zelfstandige apps (HTML/JS/SVG in een streng afgeschermde sandbox).
Knoppen en weergaveschakelaars in een widget werken onmiddellijk — voor het wisselen
van een diagramweergave is de agent nooit nodig.

## Het bord

- **Flexibel raster.** Sleep widgets aan hun handgreep; alles herschikt en
  compacteert automatisch. Wijzig de grootte met de handgreep of kies een
  vooraf ingestelde grootte (klein, middelgroot, groot, extra groot) in het
  widgetmenu. Niemand plaatst pixels — jij niet en de agent niet.
- **Tabbladen.** Een bord kan meerdere pagina's hebben — bijvoorbeeld een
  overzichtstabblad en een gericht tabblad met één grote widget. Elk tabblad
  onthoudt zijn eigen positie van het vastgezette chatvenster.
- **Vastgezette chat.** In de dashboardweergave wordt je gesprek links, rechts
  of onderaan vastgezet, kan het net als de zijbalk van grootte worden gewijzigd
  en kan het volledig worden verborgen — de agent hoort je nog steeds wanneer
  je het weer tevoorschijn haalt.
- **Gelijke mogelijkheden voor de agent.** Alles wat jij kunt doen, kan de agent doen met zijn
  tool `dashboard`: widgets toevoegen, bijwerken, verplaatsen, vergroten,
  verkleinen en verwijderen, tabbladen beheren, het zichtbare tabblad wijzigen
  en het chatvenster verplaatsen of verbergen. Vraag "zet de chat links en toon
  het tabblad financiën" en zie het gebeuren.

## Wat widgets mogen doen

Een widget die alleen inhoud weergeeft, heeft geen goedkeuring nodig — deze verschijnt
direct, precies zoals inline chatwidgets, en de netwerktoegang is volledig uitgeschakeld.

Widgets die **toegang** willen, moeten dit declareren en je verleent die eenmaal per
widget met één tik:

- **Netwerk** (`net`): haal gedeclareerde HTTPS-oorsprongen rechtstreeks op vanuit de sandbox —
  bijvoorbeeld een weerkaart die zichzelf vernieuwt via een API.
- **Gateway-gegevens** (`data`): alleen-lezenfeeds zoals sessies, gebruik of
  cronstatus, verwerkt door de Gateway — de widget bevat nooit je token.
- **Automatisering** (`actions`): activeer een specifieke crontaak, zodat een knop
  een echte taak kan uitvoeren (die mogelijk een kleiner model gebruikt) zonder
  je hoofdgesprek te activeren.
- **Prompt** (`prompt`): stuur berichten naar je thread zonder de bevestiging
  per klik die voor niet-goedgekeurde widgets vereist is.

Ingeschakelde plugins kunnen hun eigen benoemde alleen-lezenfeeds en acties aan deze mogelijkhedenlijsten toevoegen; als je de plugin uitschakelt, worden die integraties verwijderd.

Toestemmingen zijn gekoppeld aan de exacte widgetbytes en revisie die je hebt beoordeeld.
Als de agent de widget wijzigt en om _meer_ vraagt dan je hebt goedgekeurd, krijgt deze
weer de status in behandeling; als inhoud binnen dezelfde machtigingen wordt vernieuwd,
blijft de toestemming behouden. Widgetinteracties waarvan de agent op de hoogte moet zijn
(filters waarop je hebt geklikt, weergaven waarnaar je bent overgeschakeld) bereiken deze
ongemerkt als sessiemeldingen — de agent blijft op de hoogte zonder te worden onderbroken.

## MCP-apps op het bord

Als op je Gateway MCP-servers zijn geconfigureerd, kunnen interactieve MCP-apps die in
de chat verschijnen net als elke widget worden vastgemaakt. Vastgemaakte apps komen op
het bord met nieuwe sessies weer tot leven; standaard kunnen ze alleen inhoud weergeven.
Als je de widget toegang tot de gedeclareerde servertools verleent, wordt deze volledig
interactief — met dezelfde goedkeuring met één tik, gekoppeld aan de revisie, als voor al
het andere.

## Goed om te weten

- Bij het resetten van een thread met een bord wordt om bevestiging gevraagd en
  blijft het bord behouden.
- Als je een thread verwijdert, wordt het bijbehorende bord verwijderd.
- Borden bevinden zich op je Gateway (in de database van de agent die de eigenaar is)
  en verschijnen op elk apparaat waarmee je verbinding maakt.
- Het beveiligingsmodel, de opslagdetails en de ontwerpoverwegingen staan in
  [Dashboardarchitectuur](/web/dashboard-architecture), inclusief de
  gedocumenteerde afwegingen van de sandbox.
