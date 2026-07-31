---
doc-schema-version: 1
read_when:
    - Je wilt dat OpenClaw gedurende een lange sessie één doelstelling zichtbaar houdt
    - Je moet een sessiedoel pauzeren, hervatten, blokkeren, voltooien of wissen
    - Je wilt de tools get_goal, create_goal en update_goal begrijpen
    - Je wilt zien hoe doelen in de TUI worden weergegeven
summary: 'Sessiedoelen: duurzame doelstellingen per sessie, /goal-besturingselementen, modeltools voor doelen, tokenbudgetten en TUI-status'
title: Doel
x-i18n:
    generated_at: "2026-07-27T06:15:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8bfe25eb9901394b32b61729fbcb6a7bd711ed859d284fa39b637000ed7f0a18
    source_path: tools/goal.md
    workflow: 16
---

# Doel

Een **doel** is één duurzame doelstelling die aan de huidige OpenClaw-sessie is gekoppeld.
Het biedt de agent en de operator een gezamenlijk doel voor langdurig werk,
zonder dat doel om te zetten in een achtergrondtaak, herinnering, Cron-taak of
doorlopende opdracht.

Doelen zijn sessiestatus: ze gaan mee met de sessiesleutel, blijven behouden na het
herstarten van processen en verschijnen in `/goal`, de doeltools voor het model en de
voettekst van de TUI.

Voltooide losgekoppelde opdrachten keren terug naar de oorspronkelijke gebruikersgerichte thread, zodat
de volgende beurt hetzelfde doel blijft zien, zelfs wanneer voor de uitvoering van de opdracht
een afzonderlijke sessie met sandboxbeleid is gebruikt.

## Snel aan de slag

```text
/goal start maak CI groen voor PR 87469 en push de oplossing
/goal
/goal edit maak CI groen voor PR 87469, push de oplossing en werk de documentatie bij
/goal pause wachten op CI
/goal resume
/goal complete gepusht en geverifieerd
/goal clear
```

`start` is optioneel: `/goal get CI green for PR 87469` maakt ook een doel aan,
omdat alle tekst na `/goal` die geen bekend actiewoord is, als een
nieuwe doelstelling wordt behandeld.

## Waarvoor doelen dienen

Gebruik een doel wanneer een sessie een concreet resultaat heeft dat gedurende
veel beurten zichtbaar moet blijven:

- Een PR afronden: oplossen, verifiëren, automatisch reviewen, pushen en de PR openen of bijwerken.
- Een debugronde: de bug reproduceren, het verantwoordelijke oppervlak identificeren, een patch aanbrengen en
  de oplossing aantonen.
- Een documentatieronde: de relevante documentatie lezen, de nieuwe pagina schrijven, kruisverwijzingen toevoegen en
  de documentatiebuild verifiëren.
- Een onderhoudstaak: de huidige status inspecteren, begrensde wijzigingen aanbrengen, de
  juiste controles uitvoeren en rapporteren wat er is gewijzigd.

Een doel is geen takenwachtrij. Gebruik [Task Flow](/nl/automation/taskflow),
[taken](/nl/automation/tasks), [Cron-taken](/nl/automation/cron-jobs) of
[doorlopende opdrachten](/nl/automation/standing-orders) wanneer werk losgekoppeld moet worden uitgevoerd,
volgens een planning moet worden herhaald, moet worden opgesplitst in beheerd deelwerk of als beleid moet blijven bestaan.

## Opdrachtenoverzicht

`/goal` zonder argumenten toont de samenvatting van het huidige doel:

```text
Doel
Status: actief
Doelstelling: maak CI groen voor PR 87469 en push de oplossing
Gebruikte tokens: 12k
Tokenbudget: 12k/50k

Opdrachten: /goal edit <objective>, /goal pause, /goal complete, /goal clear
```

| Opdracht                                            | Effect                                                                   |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `/goal` of `/goal status`                           | Toon het huidige doel.                                                   |
| `/goal start <objective>`                           | Maak een nieuw doel voor de huidige sessie.                               |
| `/goal set <objective>`, `/goal create <objective>` | Aliassen voor `start`.                                                     |
| `/goal <objective>`                                 | Maakt ook een nieuw doel aan (alle tekst die geen herkend actiewoord is). |
| `/goal edit <objective>`                            | Herformuleer de huidige doelstelling; de status en tokentelling blijven behouden.      |
| `/goal pause [note]`                                | Pauzeer een actief doel.                                                    |
| `/goal resume [note]`                               | Hervat een gepauzeerd, geblokkeerd, gebruiksbeperkt of budgetbeperkt doel.         |
| `/goal complete [note]`                             | Markeer het doel als behaald.                                                  |
| `/goal done [note]`                                 | Alias voor `complete`.                                                    |
| `/goal block [note]`                                | Markeer het doel als geblokkeerd.                                                   |
| `/goal blocked [note]`                              | Alias voor `block`.                                                       |
| `/goal clear`                                       | Verwijder het doel uit de sessie.                                        |

Er kan slechts één doel tegelijk in een sessie bestaan. Het starten van een tweede doel mislukt
met `Goal error: goal already exists` totdat het huidige doel is gewist.

`/goal start` accepteert geen vlag voor een tokenbudget; een budget kan alleen worden ingesteld
via de modelgerichte tool `create_goal`.

## Statussen

- `active`: de sessie werkt aan het doel.
- `paused`: de operator heeft het doel gepauzeerd; `/goal resume` maakt het weer
  actief.
- `blocked`: de agent of operator heeft een echte blokkade gemeld; `/goal resume`
  maakt het weer actief wanneer nieuwe informatie of een nieuwe status beschikbaar is.
- `budget_limited`: het geconfigureerde tokenbudget is bereikt; `/goal resume`
  hervat het nastreven van dezelfde doelstelling met een nieuw budgetvenster.
- `usage_limited`: gereserveerd voor een toekomstige stopstatus wegens een gebruikslimiet; `/goal
resume` hervat het nastreven op dezelfde manier.
- `complete`: het doel is behaald. Voltooide doelen zijn definitief; gebruik `/goal
clear` voordat je een ander doel start.

`/new` en `/reset` wissen het huidige sessiedoel, omdat ze bewust
met een nieuwe sessiecontext beginnen.

## Tokenbudgetten

Doelen kunnen een optioneel positief tokenbudget hebben, dat via de parameter
`token_budget` van de tool `create_goal` wordt ingesteld. Het budget wordt gemeten vanaf het
actuele aantal tokens van de sessie op het moment dat het doel wordt aangemaakt. Als de sessie bij
het starten van het doel alleen een verouderde of onbekende tokenmomentopname heeft, wacht OpenClaw op de
volgende actuele momentopname en gebruikt die als uitgangswaarde, zodat tokens die zijn besteed voordat het
doel bestond er niet aan worden toegerekend.

Wanneer het gebruik het budget bereikt, krijgt het doel de status `budget_limited`. Hierdoor wordt
het doel niet verwijderd en de doelstelling niet gewist; het informeert de operator en de
agent dat het doel niet langer actief wordt nagestreefd totdat het wordt hervat of
gewist. Bij hervatten begint een nieuw budgetvenster bij het huidige actuele
aantal tokens.

Tokenbudgetten zijn een vangrail voor sessiedoelen, geen factureringslimiet. Providerquota,
kostenrapportage en het gedrag van het contextvenster blijven de normale
gebruiks- en modelinstellingen van OpenClaw volgen.

## Modeltools

OpenClaw stelt drie doeltools beschikbaar aan agentharnassen:

| Tool          | Doel                                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `get_goal`    | Lees het huidige sessiedoel: status, doelstelling, tokengebruik en tokenbudget.                                         |
| `create_goal` | Maak alleen een doel aan wanneer de gebruiker of systeeminstructies daar expliciet om vragen. Mislukt als de sessie al een doel heeft. |
| `update_goal` | Markeer het doel als `complete` of `blocked`.                                                                                   |

Het model kan een doel niet stilzwijgend pauzeren, hervatten, wissen of vervangen. Dat blijven
operator- en sessiebesturingselementen via `/goal` en resetopdrachten, zodat de agent
een behaald resultaat of een echte blokkade kan melden zonder ongemerkt het
doel te wijzigen.

`update_goal` mag een doel alleen als `complete` markeren wanneer de doelstelling
daadwerkelijk is behaald. Een doel mag pas als `blocked` worden gemarkeerd nadat dezelfde
blokkerende omstandigheid zich gedurende ten minste drie opeenvolgende doelbeurten heeft herhaald, niet vanwege
gewone moeilijkheden of ontbrekende afwerking.

## Doelcontext bij elke beurt

Elke gebruikers-/chatbeurt met een actief doel bevat deze contextregel met de gebruikersrol:

```text
Actief doel: <objective> — werk eraan verder of werk de status bij (get_goal/update_goal).
```

OpenClaw houdt de regel compact door lange doelstellingen af te kappen. Gepauzeerde,
geblokkeerde, budgetbeperkte, gebruiksbeperkte en voltooide doelen worden niet ingevoegd,
zodat een stop door de operator van kracht blijft totdat het doel wordt hervat.

## Control UI

De webgebaseerde Control UI toont het doel als een compacte pil boven het chatinvoerveld:
een statuspictogram, het statuslabel (bijvoorbeeld `Pursuing goal`), de afgekorte
doelstelling en een live timer voor de verstreken tijd.

De pil bevat ingebouwde bedieningselementen:

- **Potlood** vult het invoerveld vooraf met `/goal edit <objective>`, zodat de
  doelstelling opnieuw kan worden geformuleerd en verzonden.
- **Pauzeren / hervatten** schakelt afhankelijk van de huidige status tussen `/goal pause` en `/goal resume`.
- **Prullenbak** verzendt `/goal clear`.
- **Chevron** vouwt de pil uit om de volledige doelstelling, de nieuwste statusnotitie,
  het tokengebruik en de verstreken tijd weer te geven.

De actieknoppen zijn verborgen zolang het invoerveld niets kan verzenden (bijvoorbeeld
wanneer de verbinding met de Gateway is verbroken); de uitvouwchevron blijft werken.

## TUI

De voettekst van de TUI houdt het doel van de actieve sessie zichtbaar naast de velden voor de agent,
sessie en het model, vóór de token-/modusindicatoren.

Voorbeelden van voetteksten:

- `Pursuing goal (12k/50k)` voor een actief doel met een tokenbudget.
- `Goal paused (/goal resume)` voor een gepauzeerd doel.
- `Goal blocked (/goal resume)` voor een geblokkeerd doel.
- `Goal hit usage limits (/goal resume)` voor een gebruiksbeperkt doel.
- `Goal unmet (50k/50k)` voor een budgetbeperkt doel.
- `Goal achieved (42k)` voor een voltooid doel.

De voettekst is bewust compact. Gebruik `/goal` voor de volledige doelstelling,
notitie, het tokenbudget en de beschikbare opdrachten.

## Kanaalgedrag

`/goal` werkt in OpenClaw-sessies die opdrachten ondersteunen, waaronder de TUI en
chatoppervlakken die tekstopdrachten toestaan. De doelstatus is gekoppeld aan de
sessiesleutel, niet aan het transport, zodat twee oppervlakken die een sessiesleutel delen hetzelfde
doel zien.

De doelstatus is geen bezorginstructie: deze dwingt geen antwoorden via een
kanaal af, wijzigt het gedrag van de wachtrij niet, keurt geen tools goed en plant geen werk.

## Problemen oplossen

| Bericht                                | Betekenis                                                                                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Goal error: goal already exists`      | De sessie heeft al een doel. Gebruik `/goal` om het te bekijken, `/goal complete` als het klaar is of `/goal clear` voordat je een andere doelstelling start. |
| `Goal error: goal not found`           | De sessie heeft nog geen doel. Start er een met `/goal start <objective>`.                                                                       |
| `Goal error: goal is already complete` | Het doel is definitief. Wis het voordat je een andere doelstelling start of hervat.                                                                |

Als het tokengebruik `0` weergeeft of verouderd lijkt, heeft de actieve sessie mogelijk nog geen
actuele tokenmomentopname. Het gebruik wordt bijgewerkt wanneer OpenClaw sessiegebruik
en uit het transcript afgeleide totalen registreert.

## Gerelateerd

- [Slash-opdrachten](/nl/tools/slash-commands)
- [TUI](/nl/web/tui)
- [Sessietool](/nl/concepts/session-tool)
- [Compaction](/nl/concepts/compaction)
- [Task Flow](/nl/automation/taskflow)
- [Doorlopende opdrachten](/nl/automation/standing-orders)
