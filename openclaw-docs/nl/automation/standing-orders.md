---
read_when:
    - Autonome agentworkflows instellen die zonder prompts per taak worden uitgevoerd
    - Bepalen wat de agent zelfstandig kan doen en waarvoor menselijke goedkeuring nodig is
    - Multi-programma-agents structureren met duidelijke grenzen en escalatieregels
summary: Definieer permanente operationele bevoegdheid voor autonome agentprogramma's
title: Doorlopende opdrachten
x-i18n:
    generated_at: "2026-07-27T05:42:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e7ad622efe734facc9dc3716f5ee7f57ed3923499db78730bda234a5c62ad80
    source_path: automation/standing-orders.md
    workflow: 16
---

Vaste opdrachten geven je agent **permanente operationele bevoegdheid** voor gedefinieerde programma's. In plaats van de agent voor elke taak een prompt te geven, definieer je programma's met een duidelijke reikwijdte, triggers en escalatieregels, waarna de agent zelfstandig binnen die grenzen handelt: "Jij bent verantwoordelijk voor het wekelijkse rapport. Stel het elke vrijdag samen, verstuur het en escaleer alleen als iets niet in orde lijkt."

## Waarom vaste opdrachten

**Zonder vaste opdrachten:** je geeft de agent voor elke taak een prompt, routinewerk wordt vergeten of vertraagd en jij wordt de bottleneck.

**Met vaste opdrachten:** de agent handelt zelfstandig binnen gedefinieerde grenzen, routinewerk wordt volgens planning uitgevoerd en je wordt alleen betrokken bij uitzonderingen en goedkeuringen.

## Hoe ze werken

Vaste opdrachten worden gedefinieerd in de bestanden van je [agentwerkruimte](/nl/concepts/agent-workspace). De aanbevolen aanpak is om ze rechtstreeks op te nemen in `AGENTS.md` (dat elke sessie automatisch wordt geïnjecteerd), zodat de agent ze altijd in de context heeft. Voor grotere configuraties kun je ze ook in een afzonderlijk bestand plaatsen, zoals `standing-orders.md`, en daarnaar verwijzen vanuit `AGENTS.md`.

Elk programma specificeert:

1. **Reikwijdte** - waartoe de agent bevoegd is
2. **Triggers** - wanneer het moet worden uitgevoerd (planning, gebeurtenis of voorwaarde)
3. **Goedkeuringsmomenten** - waarvoor menselijke goedkeuring nodig is voordat er wordt gehandeld
4. **Escalatieregels** - wanneer de agent moet stoppen en om hulp moet vragen

De agent laadt deze instructies elke sessie via de bootstrapbestanden van de werkruimte (zie [Agentwerkruimte](/nl/concepts/agent-workspace) voor de volledige lijst met automatisch geïnjecteerde bestanden) en voert ze uit, in combinatie met [Cron-taken](/nl/automation/cron-jobs) voor tijdgebonden handhaving.

<Tip>
Plaats vaste opdrachten in `AGENTS.md` om te garanderen dat ze elke sessie worden geladen. De bootstrap van de werkruimte injecteert automatisch `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` en `MEMORY.md`, maar geen willekeurige bestanden in submappen.
</Tip>

## Opbouw van een vaste opdracht

```markdown
## Programma: Wekelijks statusrapport

**Bevoegdheid:** Gegevens verzamelen, rapport genereren, aan belanghebbenden leveren
**Trigger:** Elke vrijdag om 16.00 uur (afgedwongen via een Cron-taak)
**Goedkeuringsmoment:** Geen voor standaardrapporten. Markeer afwijkingen voor menselijke beoordeling.
**Escalatie:** Als een gegevensbron niet beschikbaar is of meetwaarden ongebruikelijk lijken (>2σ van de norm)

### Uitvoeringsstappen

1. Haal meetwaarden op uit geconfigureerde bronnen
2. Vergelijk ze met de vorige week en de doelstellingen
3. Genereer het rapport in Reports/weekly/YYYY-MM-DD.md
4. Lever de samenvatting via het geconfigureerde kanaal
5. Registreer de voltooiing in Agent/Logs/

### Wat je NIET moet doen

- Stuur geen rapporten naar externe partijen
- Wijzig geen brongegevens
- Sla de levering niet over als meetwaarden slecht lijken; rapporteer nauwkeurig
```

## Vaste opdrachten in combinatie met Cron-taken

Vaste opdrachten definiëren **wat** de agent mag doen. [Cron-taken](/nl/automation/cron-jobs) definiëren **wanneer** het gebeurt. Ze werken samen:

```text
Vaste opdracht: "Jij bent verantwoordelijk voor de dagelijkse triage van het Postvak IN"
    ↓
Cron-taak (dagelijks om 08.00 uur): "Voer de triage van het Postvak IN uit volgens de vaste opdrachten"
    ↓
Agent: Leest vaste opdrachten → voert stappen uit → rapporteert resultaten
```

De prompt voor de Cron-taak moet naar de vaste opdracht verwijzen in plaats van deze te dupliceren:

```bash
openclaw cron add \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel imessage \
  --to "+1XXXXXXXXXX" \
  --message "Voer de dagelijkse triage van het Postvak IN uit volgens de vaste opdrachten. Controleer de e-mail op nieuwe waarschuwingen. Analyseer, categoriseer en bewaar elk item. Rapporteer een samenvatting aan de eigenaar. Escaleer onbekende gevallen."
```

## Voorbeelden

### Voorbeeld 1: content en sociale media (wekelijkse cyclus)

```markdown
## Programma: Content en sociale media

**Bevoegdheid:** Content opstellen, berichten inplannen, rapporten over betrokkenheid samenstellen
**Goedkeuringsmoment:** Alle berichten moeten gedurende de eerste 30 dagen door de eigenaar worden beoordeeld; daarna geldt permanente goedkeuring
**Trigger:** Wekelijkse cyclus (beoordeling op maandag → concepten halverwege de week → samenvatting op vrijdag)

### Wekelijkse cyclus

- **Maandag:** Beoordeel platformmeetwaarden en betrokkenheid van het publiek
- **Dinsdag-donderdag:** Stel berichten voor sociale media op en maak blogcontent
- **Vrijdag:** Stel de wekelijkse marketingsamenvatting samen → lever deze aan de eigenaar

### Contentregels

- De schrijfstijl moet bij het merk passen (zie SOUL.md of de stijlgids van het merk)
- Maak in openbare content nooit bekend dat je AI bent
- Neem meetwaarden op wanneer die beschikbaar zijn
- Richt je op waarde voor het publiek, niet op zelfpromotie
```

### Voorbeeld 2: financiële activiteiten (gestuurd door gebeurtenissen)

```markdown
## Programma: Financiële verwerking

**Bevoegdheid:** Transactiegegevens verwerken, rapporten genereren, samenvattingen versturen
**Goedkeuringsmoment:** Geen voor analyses. Aanbevelingen vereisen goedkeuring van de eigenaar.
**Trigger:** Nieuw gegevensbestand gedetecteerd OF geplande maandelijkse cyclus

### Wanneer nieuwe gegevens binnenkomen

1. Detecteer een nieuw bestand in de aangewezen invoermap
2. Analyseer en categoriseer alle transacties
3. Vergelijk ze met de budgetdoelstellingen
4. Markeer: ongebruikelijke posten, overschrijdingen van drempelwaarden, nieuwe terugkerende kosten
5. Genereer een rapport in de aangewezen uitvoermap
6. Lever de samenvatting via het geconfigureerde kanaal aan de eigenaar

### Escalatieregels

- Afzonderlijke post > $500: onmiddellijke waarschuwing
- Categorie > 20% boven budget: markeer dit in het rapport
- Onherkenbare transactie: vraag de eigenaar om categorisering
- Verwerking mislukt na 2 nieuwe pogingen: rapporteer de fout, ga niet raden
```

### Voorbeeld 3: bewaking en waarschuwingen (continu)

```markdown
## Programma: Systeembewaking

**Bevoegdheid:** Systeemstatus controleren, services opnieuw starten, waarschuwingen versturen
**Goedkeuringsmoment:** Start services automatisch opnieuw. Escaleer als het opnieuw starten tweemaal mislukt.
**Trigger:** Elke Heartbeat-cyclus

### Controles

- Statusendpoints van services reageren
- Schijfruimte boven de drempelwaarde
- Openstaande taken zijn niet verouderd (>24 uur)
- Leveringskanalen zijn operationeel

### Reactiematrix

| Voorwaarde         | Actie                              | Escaleren?                          |
| ------------------ | ---------------------------------- | ----------------------------------- |
| Service uitgevallen | Automatisch opnieuw starten        | Alleen als opnieuw starten 2x mislukt |
| Schijfruimte < 10% | Waarschuw de eigenaar              | Ja                                  |
| Verouderde taak > 24h | Herinner de eigenaar            | Nee                                 |
| Kanaal offline     | Registreer en probeer volgende cyclus opnieuw | Als het > 2 uur offline is |
```

## Patroon uitvoeren-verifiëren-rapporteren

Vaste opdrachten werken het beste in combinatie met strikte uitvoeringsdiscipline. Elke taak in een vaste opdracht moet deze cyclus volgen:

1. **Uitvoeren** - Voer het daadwerkelijke werk uit (bevestig de instructie niet alleen)
2. **Verifiëren** - Bevestig dat het resultaat correct is (bestand bestaat, bericht is geleverd, gegevens zijn geanalyseerd)
3. **Rapporteren** - Vertel de eigenaar wat er is gedaan en wat er is geverifieerd

```markdown
### Uitvoeringsregels

- Elke taak volgt Uitvoeren-Verifiëren-Rapporteren. Zonder uitzonderingen.
- "Ik ga dat doen" is geen uitvoering. Doe het en rapporteer daarna.
- "Klaar" zonder verificatie is niet acceptabel. Bewijs het.
- Als de uitvoering mislukt: probeer het eenmaal opnieuw met een aangepaste aanpak.
- Als het nog steeds mislukt: rapporteer de fout met een diagnose. Laat fouten nooit onvermeld.
- Blijf nooit onbeperkt opnieuw proberen: maximaal 3 pogingen, daarna escaleren.
```

Dit patroon voorkomt de meest voorkomende foutmodus van agents: een taak bevestigen zonder deze te voltooien.

## Architectuur met meerdere programma's

Voor agents die meerdere aandachtsgebieden beheren, organiseer je vaste opdrachten als afzonderlijke programma's met duidelijke grenzen:

```markdown
## Programma 1: [Domein A] (Wekelijks)

...

## Programma 2: [Domein B] (Maandelijks + op aanvraag)

...

## Programma 3: [Domein C] (Wanneer nodig)

...

## Escalatieregels (Alle programma's)

- [Algemene escalatiecriteria]
- [Goedkeuringsmomenten die voor alle programma's gelden]
```

Elk programma moet het volgende hebben:

- Een eigen **triggerfrequentie** (wekelijks, maandelijks, gebeurtenisgestuurd, continu)
- Eigen **goedkeuringsmomenten** (sommige programma's vereisen meer toezicht dan andere)
- Duidelijke **grenzen** (de agent moet weten waar het ene programma eindigt en het andere begint)

## Aanbevolen werkwijzen

### Wel doen

- Begin met beperkte bevoegdheid en breid deze uit naarmate het vertrouwen groeit
- Definieer expliciete goedkeuringsmomenten voor acties met een hoog risico
- Neem secties met "Wat je NIET moet doen" op; grenzen zijn net zo belangrijk als bevoegdheden
- Combineer met Cron-taken voor betrouwbare tijdgebonden uitvoering
- Beoordeel agentlogboeken wekelijks om te verifiëren dat vaste opdrachten worden gevolgd
- Werk vaste opdrachten bij wanneer je behoeften veranderen; het zijn levende documenten

### Vermijden

- Verleen op de eerste dag brede bevoegdheid ("doe wat jij het beste vindt")
- Sla escalatieregels niet over; elk programma heeft een bepaling nodig voor wanneer de agent moet stoppen en om hulp moet vragen
- Ga er niet van uit dat de agent mondelinge instructies onthoudt; zet alles in het bestand
- Combineer geen aandachtsgebieden in één programma; gebruik afzonderlijke programma's voor afzonderlijke domeinen
- Vergeet handhaving met Cron-taken niet; vaste opdrachten zonder triggers worden suggesties

## Gerelateerd

- [Automatisering](/nl/automation): alle automatiseringsmechanismen in één oogopslag.
- [Cron-taken](/nl/automation/cron-jobs): geplande handhaving van vaste opdrachten.
- [Hooks](/nl/automation/hooks): gebeurtenisgestuurde scripts voor gebeurtenissen in de levenscyclus van agents.
- [Webhooks](/nl/automation/cron-jobs#webhooks): triggers voor inkomende HTTP-gebeurtenissen.
- [Agentwerkruimte](/nl/concepts/agent-workspace): waar vaste opdrachten worden opgeslagen, inclusief de volledige lijst met automatisch geïnjecteerde bootstrapbestanden (`AGENTS.md`, `SOUL.md`, enzovoort).
