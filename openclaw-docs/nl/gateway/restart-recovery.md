---
read_when:
    - Je wilt weten of het herstarten van de Gateway lopend agentwerk verloren laat gaan
    - Een agentuitvoering is onderbroken door een herstart, crash of het opnieuw laden van de configuratie
    - Je debugt automatisch sessieherstel nadat de Gateway weer actief is
summary: 'Wat een herstart of crash van de Gateway overleeft: onderbroken agentbeurten worden automatisch hervat, subagents en achtergrondtaken worden hersteld, leveringen in de wachtrij worden afgehandeld'
title: Herstel na opnieuw opstarten
x-i18n:
    generated_at: "2026-07-27T05:05:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdea30f3a90697951f4f63a06897d2c1d936e5145138b47fed7d8ebd8b7187ad
    source_path: gateway/restart-recovery.md
    workflow: 16
---

Door de Gateway opnieuw te starten gaat de status van agents niet verloren. Gesprekken, transcripties,
geplande taken, registraties van achtergrondtaken en in de wachtrij geplaatste uitgaande berichten
worden allemaal op schijf opgeslagen, en werk dat halverwege een beurt werd onderbroken, wordt
automatisch gedetecteerd en hervat nadat de Gateway weer actief is. Herstel is altijd ingeschakeld en
vereist normaal gesproken geen handmatige tussenkomst. Herstel dat herhaaldelijk mislukt, is begrensd
en kan één sessie in quarantaine plaatsen totdat je deze inspecteert of vervangt.

Deze pagina beschrijft wat behouden blijft na een herstart, hoe onderbroken werk wordt gedetecteerd
en hoe automatisch hervatten eruitziet.

## Wat behouden blijft na een herstart

| Status                        | Opslag                                      | Gedrag bij herstart                                                      |
| ----------------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| Gespreksgeschiedenis          | SQLite-database per agent                   | Ongewijzigd; sessies gaan verder vanaf de opgeslagen transcriptie        |
| Onderbroken beurt in hoofdsessie | SQLite-sessierij en transcriptie per agent | Wordt enkele seconden na het opstarten automatisch hervat of afgestemd |
| Subagentruns                  | SQLite (gedeelde statusdatabase)            | Register wordt bij het opstarten hersteld; onderbroken runs worden hervat |
| Achtergrondtaken              | SQLite (gedeelde statusdatabase)            | Worden bij het opstarten afgestemd; verweesde runs worden hersteld of als verloren gemarkeerd |
| Uitgaande bezorgingen in de wachtrij | SQLite-bezorgingswachtrij             | Worden na de herstart verwerkt; niet-bezorgde antwoorden worden opnieuw geprobeerd |
| Geplande (Cron-)taken         | SQLite-Cronopslag                           | Planningen blijven behouden; de planner wordt bij het opstarten opnieuw geactiveerd |
| Voortzetting na herstart      | SQLite-herstartsentinel                     | Eenmalige vervolgactie wordt verzonden naar de sessie die om de herstart vroeg |

## Correcte herstarts worden eerst afgehandeld

Een aangevraagde herstart (`openclaw gateway restart`, een configuratiewijziging waarvoor
een herstart nodig is, of een update van de Gateway) beëindigt lopend werk niet onmiddellijk. De
Gateway accepteert geen nieuw werk meer en wacht vervolgens tot actieve agentbeurten en
achtergrondtaken zijn voltooid, tot aan een afhandelingsbudget (standaard 5 minuten). De meeste
herstarts onderbreken daarom helemaal niets.

Alleen werk dat niet binnen het afhandelingsbudget kan worden voltooid (of een run die wordt onderbroken
door een geforceerde herstart of crash) wordt afgebroken — en voordat dat gebeurt, wordt elke
betrokken sessie gemarkeerd voor herstel.

## Hoe onderbroken werk wordt gedetecteerd

Drie aanvullende mechanismen markeren sessies waarvan de beurt niet is voltooid:

- **Bij toelating van de beurt:** voor een gewone tekstbeurt in een bestaande hoofdsessie
  voegt de Gateway het gebruikersbericht toe, markeert deze de sessie als actief en registreert
  deze de herstelbezorgingsclaim in één SQLite-transactie voordat het model of de
  `before_agent_reply`-hook wordt uitgevoerd. Control UI doet dit voordat de
  `started`-bevestiging wordt geretourneerd; kanaalverzending doet dit wanneer de voorbereide beurt
  de agentrun overneemt.
  Opdrachten, bijlagen, overschrijvingen per beurt, openstaande bezorgingen, eerdere aanwijzingen voor afbreken,
  sessies die eigendom zijn van een Plugin en beurten met uitvoeringshooks behouden hun
  gespecialiseerde toelatingspaden.
  Als een `before_agent_reply`-hook is geïnstalleerd, registreert de toelating ook de fase ervan.
  Herstel speelt een hook die midden in een aanroep werd onderbroken nooit opnieuw af. Zodra een niet-afgehandelde hook
  is voltooid, registreert het controlepunt dat resultaat, maar herstel blijft gesloten mislukken
  zolang die hook actief blijft: een controlepunt kan niet aantonen dat na de herstart dezelfde
  Plugincode en configuratie zijn geladen. Afgehandelde tekstresultaten en
  stille resultaten krijgen afzonderlijke controlepunten voor deterministische afhandeling.
  Duurzame herstelclaims die door oudere versies zijn geschreven, hebben geen markering voor broneigendom
  en krijgen daarom tijdens een upgrade dezelfde gesloten hookcontrole.
- **Bij afsluiten:** tijdens het afhandelen van de herstart wordt elke sessie met een actieve run
  in de sessieopslag voorzien van een herstelmarkering voordat de run wordt
  afgebroken.
- **Bij opstarten:** de Gateway scant sessieopslagen op sessies die nog steeds
  aangeven actief te zijn, maar geen actieve eigenaar in het nieuwe proces hebben. Hiermee worden
  harde crashes en beëindigingen opgevangen waarbij geen afsluitcode is uitgevoerd. Verouderde vergrendelingsbestanden
  van transcripties worden tegelijkertijd opgeschoond.

## Automatisch hervatten

Enkele seconden na het opstarten verzendt de Gateway elke gemarkeerde sessie opnieuw
met een synthetisch systeembericht dat de agent vertelt dat de vorige beurt door
een herstart is onderbroken en dat deze verder moet gaan vanaf de bestaande transcriptie. Als een
definitief antwoord al was geproduceerd maar niet bezorgd, wordt de tekst ervan opgenomen,
zodat de agent het kan bezorgen in plaats van het werk opnieuw uit te voeren.

Afstemming bij het opstarten probeert tijdelijke fouten maximaal drie keer opnieuw met
exponentiële vertraging. Daarnaast heeft elke onderbroken hoofdsessiecyclus een
duurzaam budget van drie aangerekende automatische verzendpogingen, dat behouden blijft tussen
herstarts van de Gateway. OpenClaw rekent een poging aan vóór verzending, crediteert deze wanneer
de Gateway het verzoek expliciet afwijst voordat het wordt geaccepteerd en behoudt de
aanrekening wanneer een resultaat na verzending onzeker is, om te voorkomen dat werk opnieuw wordt uitgevoerd.
Voorgrondwerk dat al eigenaar is van de sessie, houdt automatisch herstel tegen
totdat dat werk is afgehandeld.

Nadat het duurzame budget is uitgeput, krijgt de sessie een tombstone in plaats van
eindeloos te blijven herhalen. Inspecteer de mislukte sessie en gebruik `/new` of `/reset` om een
vervangende sessie te starten. `openclaw doctor --fix` kan een verouderde afbreekvlag herstellen die
in strijd is met een tombstone, maar schakelt die herstelcyclus niet opnieuw in.

Elke nieuwe poging gebruikt dezelfde duurzame verzendings-ID, zodat een onduidelijke verbindingsfout
hetzelfde herstel niet tweemaal kan starten. Voltooide en niet-hervatbare Control
UI-beurten behouden ook begrensde duurzame idempotentietombstones, zodat een
opnieuw verbindende outbox ze kan verwijderen zonder het verzoek opnieuw uit te voeren.

Antwoorden die uitsluitend het berichtentool gebruiken, gebruiken een tweede duurzame correlatie. Voordat een terminale
verzending binnen hetzelfde gesprek het kanaal bereikt, registreert de Gateway een onopgeloste
bezorgingsintentie voor de exacte sessie en bronbeurt. Een bevestigd succes van de provider
zet deze om in een duurzaam bezorgd ontvangstbewijs; een bevestigde fout wist
de intentie. Herstel voltooit een bezorgd ontvangstbewijs zonder tools opnieuw uit te voeren. Als een crash
de uitkomst bij de provider onbekend laat, mislukt het herstel gesloten in plaats van
een extern effect opnieuw uit te voeren.

Het bezorgde antwoord wordt ook met de bronbericht-ID naar de transcriptie gespiegeld.
Terminale spiegelingen gebruiken een afzonderlijke ontvangstbewijssleutel, zodat een voortgangsverzending met
dezelfde idempotentiesleutel van de provider de terminale markering niet kan verbergen. Voortgangsverzendingen
en ontvangstbewijzen van oudere beurten kunnen de huidige beurt niet voltooien. Alleen
duurzame claims voor binnenkomende kanaalberichten kunnen de bevoegdheid voor berichtacties herstellen. Een hervatte
run behoudt de oorspronkelijke bronbezorgingsmodus en broncorrelatie, inclusief
de identiteit van de aanvrager en eventuele beperkingen tot hetzelfde kanaal of dezelfde thread, zodat hetzelfde ontvangstbewijs
gezaghebbend blijft, zelfs als tijdens het herstel opnieuw een herstart plaatsvindt. Een
beurt die uitsluitend het berichtentool gebruikt en geen reconstrueerbare kanaalbevoegdheid heeft, mislukt
gesloten en ontvangt de eenmalige melding om opnieuw te verzenden.

Voordat het hervatten begint, controleert de Gateway of het einde van de transcriptie veilig is om
vanaf verder te gaan. Als dit niet zo is (bijvoorbeeld wanneer de beurt eindigde met een verouderde openstaande
goedkeuring), wordt de sessie niet blind opnieuw uitgevoerd; de agent plaatst in plaats daarvan een korte
melding waarin de gebruiker wordt gevraagd het laatste verzoek opnieuw te verzenden. Voor WebChat wordt die melding
rechtstreeks naar de sessiegeschiedenis geschreven, zodat deze na opnieuw verbinden zichtbaar blijft.

OpenClaw kan ook onderbroken alleen-lezenwerk in [Code Mode](/nl/tools/code-mode)
reconstrueren. Code Mode markeert deze runs als veilig voor herstarts en weigert catalogustools
met neveneffecten of Pluginnaamruimten voordat ze worden uitgevoerd. Als een herstart plaatsvindt bij
het `wait`-besturingselement, reconstrueert de nieuwe Gateway de beurt vanuit de transcriptie
en dwingt deze af dat de gereconstrueerde uitvoering veilig blijft voor herstarts, zelfs als het
model die vlag weglaat of wist. De host beperkt de volledige gereconstrueerde
beurt tot gecontroleerde alleen-lezenkerntools en expliciet veilig opnieuw afspeelbare Plugintools,
ook wanneer Code Mode na de herstart is uitgeschakeld. Werk met neveneffecten
blijft beschermd door de melding om opnieuw te verzenden, in plaats van het risico op dubbele schrijfacties te nemen.

### Subagents

Subagentruns worden opgeslagen in de gedeelde SQLite-statusdatabase, zodat het
subagentregister het proces overleeft. Bij het opstarten wordt het register hersteld en
worden onderbroken subagentsessies hervat met hun oorspronkelijke taakcontext.
Er gelden twee veiligheidsmechanismen:

- Runs die meer dan 2 uur geleden zijn onderbroken, worden voltooid in plaats van hervat, zodat
  een Gateway die 's nachts buiten werking was geen verouderd werk opnieuw activeert.
- Een sessie waarvan het herstel herhaaldelijk mislukt, krijgt als vastgelopen sessie een tombstone, zodat
  het herstel niet eindeloos kan blijven herhalen.

### Achtergrondtaken

Het [register voor achtergrondtaken](/nl/automation/tasks) wordt ondersteund door SQLite en
afgestemd bij het opstarten en met periodieke tussenpozen: duurzame resultaten die door
voltooide runs zijn geregistreerd, worden hersteld en runs waarvan het eigenaarproces is verdwenen,
worden na een respijtperiode als verloren gemarkeerd in plaats van eindeloos te blijven hangen.

### Door agents aangevraagde herstarts

Wanneer de agent zelf een herstart activeert (door een configuratiewijziging toe te passen, de
Gateway bij te werken of via een expliciet herstartverzoek), wordt vóór het afsluiten van het proces
een herstartsentinel naar SQLite geschreven. Na het opstarten plaatst de Gateway het resultaat terug
in de oorspronkelijke chat en verzendt deze een eenmalige voortzettingsbeurt, zodat de
agent precies verdergaat waar deze was gebleven, in hetzelfde kanaal en dezelfde thread.

De getypeerde SQLite-kolommen van de sentinel zijn gezaghebbend voor de afhandeling van herstarts;
de `payload_json`-waarde is uitsluitend een schaduwkopie voor opnieuw afspelen en foutopsporing. De runtime leest, schrijft
en wist de SQLite-status zonder terugval op bestanden. Tijdens de opslagomschakeling wordt bij het opstarten
en via Doctor een begrensde statusmigratie uitgevoerd om een gevalideerde
`restart-sentinel.json` te behouden die na een update door het oudere proces is achtergelaten.
De migratie verifieert de getypeerde rij en verwijdert het bronbestand voordat de normale
afhandeling van de herstart wordt voortgezet.

## Veiligheidsmechanismen en waarneembaarheid

- **Beveiliging tegen crashlussen:** 3 onjuiste opstarts binnen 5 minuten activeren een beveiliging die
  het automatisch starten van nevendiensten bij de volgende opstart onderdrukt, zodat een crashende Gateway
  zichzelf niet versterkt. Deze herstelt zodra het venster voor onjuiste opstarts is verstreken.
- **Pogingsbudget voor hoofdsessies:** drie aangerekende automatische verzendpogingen
  per onderbroken cyclus; bij uitputting krijgt die sessie een tombstone totdat deze wordt
  geïnspecteerd en vervangen.
- **Metrieken:** herstelactiviteit wordt via
  [Prometheus](/nl/gateway/prometheus) geëxporteerd als `openclaw_session_recovery_total` en
  `openclaw_session_recovery_age_seconds`.
- **Logboeken:** herstelbeslissingen worden vastgelegd onder de
  subsystemen `main-session-restart-recovery` en `subagent-interrupted-resume`.

## Wat niet wordt hervat

- Sessies die zijn uitgesloten van herstel van hoofdsessies omdat een andere eigenaar ze al
  afhandelt: subagentsessies (subagentherstel), Cron-sessies (de
  planner voert ze opnieuw uit volgens de planning) en door ACP beheerde sessies (de verbonden IDE
  of client is eigenaar van het hervatten).
- Sessies waarvan het einde van de transcriptie niet veilig kan worden voortgezet; deze krijgen de
  hierboven beschreven melding om opnieuw te verzenden in plaats van stil opnieuw te worden uitgevoerd.
- Werk dat nooit is toegelaten: berichten die tijdens het afhandelingsvenster binnenkomen, worden
  geweigerd met een expliciete herstartfout in plaats van stil in de wachtrij van een
  stoppend proces te worden geplaatst.
- Zelfstandige ingebedde beurten kunnen geen hoofdsessie overnemen waarvoor
  herstartherstel openstaat, omdat ze de levenscycluseigenaar van de Gateway niet delen.
  Voer de beurt uit via de Gateway of stel deze daar opnieuw in met `/new` of `/reset`.
