---
read_when:
    - Je wilt dat geheugenpromotie automatisch wordt uitgevoerd
    - Je wilt begrijpen wat elke Dreaming-fase doet
    - Je wilt consolidatie afstemmen zonder MEMORY.md te vervuilen
sidebarTitle: Dreaming
summary: Consolidatie van achtergrondgeheugen met lichte, diepe en REM-fasen plus een droomdagboek
title: Dreaming
x-i18n:
    generated_at: "2026-07-27T04:56:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming is het systeem voor geheugenconsolidatie op de achtergrond in `memory-core`. Het zet sterke kortetermijnsignalen om in duurzaam geheugen, terwijl het proces uitlegbaar en controleerbaar blijft.

<Note>
Dreaming is **optioneel** en standaard uitgeschakeld.
</Note>

## Wat Dreaming schrijft

- **Machinestatus** in `memory/.dreams/` (opslag voor herinneringen, fasesignalen, opnamecontrolepunten, vergrendelingen).
- **Voor mensen leesbare uitvoer** in `DREAMS.md` (of een bestaande `dreams.md`) en optionele faserapportbestanden onder `memory/dreaming/<phase>/YYYY-MM-DD.md`.

Promotie naar het langetermijngeheugen schrijft nog steeds alleen naar `MEMORY.md`.

## Fasemodel

Dreaming voert per cyclus drie samenwerkende fasen in deze volgorde uit: light -> REM -> deep. Dit zijn interne implementatiefasen, geen afzonderlijke door de gebruiker geconfigureerde modi.

| Fase  | Doel                                              | Duurzame schrijfactie |
| ----- | ------------------------------------------------- | --------------------- |
| Light | Recent kortetermijnmateriaal sorteren en klaarzetten | Nee                |
| REM   | Reflecteren op thema's en terugkerende ideeën     | Nee                   |
| Deep  | Duurzame kandidaten scoren en promoveren          | Ja (`MEMORY.md`) |

<AccordionGroup>
  <Accordion title="Light-fase">
    - Leest de recente status van kortetermijnherinneringen, dagelijkse geheugenbestanden en geredigeerde sessietranscripten wanneer die beschikbaar zijn.
    - Ontdubbelt signalen en zet kandidaatregels klaar.
    - Schrijft een beheerd `## Light Sleep`-blok wanneer de opslag inline-uitvoer bevat.
    - Registreert versterkingssignalen voor latere rangschikking in de deep-fase.
    - Schrijft nooit naar `MEMORY.md`.

  </Accordion>
  <Accordion title="REM-fase">
    - Maakt thema- en reflectiesamenvattingen van recente kortetermijnsporen.
    - Schrijft een beheerd `## REM Sleep`-blok wanneer de opslag inline-uitvoer bevat.
    - Registreert REM-versterkingssignalen die voor de rangschikking in de deep-fase worden gebruikt.
    - Schrijft nooit naar `MEMORY.md`.

  </Accordion>
  <Accordion title="Deep-fase">
    - Rangschikt kandidaten met gewogen scores en drempelpoorten (`minScore`, `minRecallCount` en `minUniqueQueries` moeten allemaal slagen).
    - Laadt fragmenten vóór het schrijven opnieuw uit actuele dagelijkse bestanden, zodat verouderde of verwijderde fragmenten worden overgeslagen.
    - Voegt gepromoveerde vermeldingen toe aan `MEMORY.md`.
    - Schrijft een `## Deep Sleep`-samenvatting naar `DREAMS.md` en optioneel `memory/dreaming/deep/YYYY-MM-DD.md`.

  </Accordion>
</AccordionGroup>

## Opname van sessietranscripten

Dreaming kan geredigeerde sessietranscripten opnemen in het Dreaming-corpus. Wanneer ze beschikbaar zijn, worden transcripten naast dagelijkse geheugensignalen en herinneringssporen aan de light-fase toegevoegd. Persoonlijke en gevoelige inhoud wordt vóór opname geredigeerd.

## Droomdagboek

Dreaming houdt een verhalend **Droomdagboek** bij in `DREAMS.md`. Nadat elke fase voldoende materiaal heeft, voert `memory-core` op de achtergrond naar beste vermogen een subagentbeurt uit en voegt het een korte dagboekvermelding toe. Hiervoor wordt het standaardruntimemodel gebruikt, tenzij `dreaming.model` is geconfigureerd. Als het geconfigureerde model niet beschikbaar is, wordt de dagboekuitvoering eenmaal opnieuw geprobeerd met het standaardsessiemodel; fouten met vertrouwen of toelatingslijsten worden niet opnieuw geprobeerd en blijven zichtbaar in de logboeken in plaats van stilzwijgend terug te vallen op een algemene dagboekvermelding.

<Note>
Het dagboek is bedoeld om door mensen te worden gelezen in de Dreams-UI, niet als bron voor promotie. Dagboek- en rapportartefacten worden uitgesloten van kortetermijnpromotie; alleen onderbouwde geheugenfragmenten komen in aanmerking voor promotie naar `MEMORY.md`.
</Note>

Er is ook een onderbouwd historisch aanvullingstraject voor controle- en herstelwerk:

<AccordionGroup>
  <Accordion title="Aanvulopdrachten">
    - `memory rem-harness --path ... --grounded` toont een voorbeeld van onderbouwde dagboekuitvoer uit historische `YYYY-MM-DD.md`-notities.
    - `memory rem-backfill --path ...` schrijft omkeerbare, onderbouwde dagboekvermeldingen naar `DREAMS.md`.
    - `memory rem-backfill --path ... --stage-short-term` zet onderbouwde duurzame kandidaten klaar in dezelfde opslag voor kortetermijnbewijs die door de normale deep-fase wordt gebruikt.
    - `memory rem-backfill --rollback` en `--rollback-short-term` verwijderen die klaargezette aanvullingsartefacten zonder gewone dagboekvermeldingen of actuele kortetermijnherinneringen te wijzigen.

  </Accordion>
</AccordionGroup>

De Control UI biedt dezelfde stroom voor aanvulling en reset van het dagboek op het tabblad Memory van de agent (pagina Agents), zodat je de resultaten in de droomscène kunt bekijken voordat je beslist of onderbouwde kandidaten promotie verdienen. Een afzonderlijk onderbouwd Scene-traject toont welke klaargezette kortetermijnvermeldingen afkomstig zijn van historische herhaling, welke gepromoveerde items voornamelijk door onderbouwde gegevens zijn bepaald, en laat je alleen klaargezette vermeldingen verwijderen die uitsluitend onderbouwd zijn, zonder de actuele kortetermijnstatus te wijzigen.

## Signalen voor rangschikking in de deep-fase

De rangschikking in de deep-fase gebruikt zes gewogen basissignalen plus faseversterking:

| Signaal               | Gewicht | Beschrijving                                                |
| --------------------- | ------- | ----------------------------------------------------------- |
| Relevantie            | 0.30    | Gemiddelde opvraagkwaliteit voor de vermelding               |
| Frequentie            | 0.24    | Hoeveel kortetermijnsignalen de vermelding heeft verzameld   |
| Querydiversiteit      | 0.15    | Afzonderlijke query-/dagcontexten waarin deze naar voren kwam |
| Recentheid            | 0.15    | In de tijd afnemende versheidsscore                          |
| Consolidatie          | 0.10    | Sterkte van herhaling over meerdere dagen                    |
| Conceptuele rijkdom   | 0.06    | Dichtheid van concepttags uit fragment/pad                   |

Treffers in de light- en REM-fase voegen een kleine, met de tijd afnemende recentheidsboost toe vanuit `memory/.dreams/phase-signals.json`.

Resultaten van schaduwproeven kunnen boven op de basisscore als controlesignaal worden toegepast voordat er duurzaam wordt geschreven: een nuttige proef geeft een kandidaat een kleine begrensde boost, een neutrale proef houdt deze uitgesteld en een schadelijke proef markeert deze als afgewezen voor die scoreronde. Dit signaal is uitsluitend voor rapportage: het kan de volgorde van kandidaten of controlemetagegevens wijzigen, maar schrijft nooit naar `MEMORY.md` en promoveert niet zelfstandig een kandidaat.

### Rapportdekking voor QA-schaduwproeven

QA Lab bevat een scenario dat uitsluitend rapporteert om te onderzoeken hoe een toekomstige Dreaming-schaduwproef een kandidaatgeheugen vóór promotie zou kunnen beoordelen: een agent vergelijkt een basisantwoord met een antwoord dat het kandidaatgeheugen kan gebruiken en schrijft vervolgens een lokaal rapport met een oordeel, reden en risicomarkeringen. Deze dekking is beperkt tot QA: ze verifieert dat het rapportartefact gescheiden blijft van `MEMORY.md` en dat de agent nooit beweert dat de kandidaat is gepromoveerd. Ze voegt geen productieschaduwproefgedrag toe en wijzigt de promotie-engine van de deep-fase niet.

De schaduwproefrunner `memory-core` behoudt hetzelfde contract van uitsluitend rapporteren voor codepaden die een stabiel artefact nodig hebben. Deze accepteert de kandidaat, proefprompt, basisuitkomst, kandidaatuitkomst, oordeel, reden, risicomarkeringen en bewijsverwijzingen en schrijft vervolgens een rapport met `promotion action: report-only`. Nuttige oordelen worden gekoppeld aan een aanbeveling voor `promote`, neutrale oordelen aan `defer` en schadelijke oordelen aan `reject`; geen daarvan schrijft naar `MEMORY.md` of past promotie in de deep-fase toe.

## Planning

Wanneer ingeschakeld, beheert `memory-core` automatisch één Cron-taak voor een volledige Dreaming-cyclus, ontdubbeld over de primaire runtimewerkruimte en alle geconfigureerde agentwerkruimten, zodat uitwaaiering van subagentwerkruimten `DREAMS.md` en de geheugenstatus van de hoofdagent niet uitsluit.

| Instelling             | Standaardmodel |
| ---------------------- | -------------- |
| `dreaming.frequency` | `0 3 * * *`   |
| `dreaming.model`     | standaardmodel |

## Snel aan de slag

<Tabs>
  <Tab title="Dreaming inschakelen">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="Aangepast cyclusritme">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## Slash-opdracht

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

`/dreaming on` en `/dreaming off` vereisen de eigenaarstatus voor kanaalaanroepers of `operator.admin` voor Gateway-clients. `/dreaming status` en `/dreaming help` zijn alleen-lezen.

## CLI-workflow

<Tabs>
  <Tab title="Promotievoorbeeld / toepassen">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    Handmatige `memory promote` gebruikt standaard de drempels van de deep-fase, tenzij deze met CLI-vlaggen worden overschreven.

  </Tab>
  <Tab title="Promotie uitleggen">
    Leg uit waarom een specifieke kandidaat wel of niet zou worden gepromoveerd:

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="Voorbeeld van REM-harnas">
    Bekijk REM-reflecties, kandidaatwaarheden en uitvoer voor promotie in de deep-fase zonder iets te schrijven:

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## Belangrijkste standaardwaarden

Alle instellingen bevinden zich onder `plugins.entries.memory-core.config.dreaming`.

<ParamField path="enabled" type="boolean" default="false">
  Schakel de Dreaming-cyclus in of uit.
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  Cron-ritme voor de volledige Dreaming-cyclus.
</ParamField>
<ParamField path="model" type="string">
  Optionele modeloverschrijving voor de Droomdagboek-subagent. Gebruik een canonieke `provider/model`-waarde wanneer je ook een `allowedModels`-toelatingslijst voor een subagent instelt.
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  Maximaal geschat aantal tokens dat wordt behouden van elk fragment uit kortetermijnherinneringen dat naar `MEMORY.md` wordt gepromoveerd. De herkomst van de rangschikking blijft zichtbaar.
</ParamField>

<Warning>
`dreaming.model` vereist `plugins.entries.memory-core.subagent.allowModelOverride: true`. Stel ook `plugins.entries.memory-core.subagent.allowedModels` in om dit te beperken. De automatische nieuwe poging geldt alleen voor fouten waarbij het model niet beschikbaar is; fouten met vertrouwen of toelatingslijsten blijven zichtbaar in de logboeken in plaats van stilzwijgend terug te vallen.
</Warning>

<Note>
Het meeste fasebeleid, de meeste drempels en het meeste opslaggedrag zijn interne implementatiedetails. Zie [Referentie voor geheugenconfiguratie](/nl/reference/memory-config#dreaming) voor de volledige lijst met sleutels.
</Note>

## Dreams-UI

Wanneer ingeschakeld, toont het tabblad **Dreams** van de Gateway:

- de huidige ingeschakelde status van Dreaming
- de status per fase en de aanwezigheid van een beheerde cyclus
- aantallen voor kortetermijngeheugen, onderbouwde gegevens, signalen en vandaag gepromoveerde items
- het tijdstip van de volgende geplande uitvoering
- een afzonderlijk onderbouwd Scene-traject voor klaargezette vermeldingen uit historische herhaling
- een uitvouwbare Droomdagboeklezer op basis van `doctor.memory.dreamDiary`

## Gerelateerd

- [Geheugen](/nl/concepts/memory)
- [Geheugen-CLI](/nl/cli/memory)
- [Referentie voor geheugenconfiguratie](/nl/reference/memory-config)
- [Zoeken in het geheugen](/nl/concepts/memory-search)
