---
read_when:
    - Je wilt begrijpen hoe Task Flow zich verhoudt tot achtergrondtaken
    - Je komt Task Flow of openclaw tasks flow tegen in releaseopmerkingen of documentatie
    - Je wilt de duurzame flowstatus inspecteren of beheren
summary: TaskFlow-orkestratielaag boven achtergrondtaken
title: Taakstroom
x-i18n:
    generated_at: "2026-07-27T06:03:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow is de orkestratielaag boven [achtergrondtaken](/nl/automation/tasks). Een flow is een duurzame registratie van werk met meerdere stappen, met een eigen status, JSON-statusgegevens, revisieteller en gekoppelde taakregistraties. Flows blijven behouden na herstarts van de Gateway; afzonderlijke taken blijven de eenheid voor losgekoppeld werk.

## Wanneer gebruik je Task Flow?

| Scenario                                  | Gebruik                                      |
| ----------------------------------------- | -------------------------------------------- |
| Eén achtergrondtaak                       | Gewone taak                                  |
| Pijplijn met meerdere stappen, aangestuurd door plugincode | Task Flow (beheerd)              |
| Losgekoppelde ACP- of subagentstart       | Task Flow (gespiegeld, automatisch gemaakt)  |
| Eenmalige herinnering                     | Cron-taak                                    |

## Synchronisatiemodi

### Beheerde modus

Een beheerde flow heeft een controller: plugincode die de flow via de Task Flow-API van de pluginruntime maakt met een doel en een vereiste controller-id, en de flow vervolgens expliciet aanstuurt.

- Elke stap wordt uitgevoerd als een achtergrondtaak die onder de flow is gemaakt; de eigenaarsleutel en oorsprong van de aanvrager van de flow worden overgenomen door onderliggende taken.
- De controller laat de flow overgaan tussen `running`, `waiting` en eindstatussen, en slaat willekeurige JSON-statusgegevens van stappen op in de flowregistratie.
- Bij elke wijziging wordt de verwachte revisie van de flow doorgegeven. Een verouderde schrijfbewerking wordt als revisieconflict geweigerd in plaats van nieuwere statusgegevens te overschrijven.
- Zodra annulering is aangevraagd, worden nieuwe onderliggende taken geweigerd en krijgt de flow de eindstatus `cancelled` wanneer er geen onderliggende taak meer actief is.

Voorbeeld: een wekelijkse rapportageflow die (1) gegevens verzamelt, (2) het rapport genereert en (3) het aflevert, met één achtergrondtaak per stap:

```
Flow: weekly-report
  Stap 1: gather-data     → taak gemaakt → geslaagd
  Stap 2: generate-report → taak gemaakt → geslaagd
  Stap 3: deliver         → taak gemaakt → wordt uitgevoerd
```

### Gespiegelde modus

OpenClaw maakt automatisch een gespiegelde flow met één taak wanneer een losgekoppelde ACP- of subagentuitvoering begint (sessiegebonden taken met opleverbare voltooiing). De flowregistratie spiegelt de ene onderliggende taak — status, doel en timing — zodat losgekoppelde starts een stabiele flowreferentie krijgen voor status- en herhaalinterfaces zonder controller. Gespiegelde flows tonen synchronisatiemodus `task_mirrored` in de CLI.

## Flowstatussen

| Status      | Betekenis                                                                  |
| ----------- | -------------------------------------------------------------------------- |
| `queued`    | Gemaakt, nog niet gestart                                                  |
| `running`   | De flow wordt actief uitgevoerd                                            |
| `waiting`   | De beheerde flow is gepauzeerd op wachtmetadata (timer, externe gebeurtenis) |
| `blocked`   | Een stap is voltooid zonder bruikbaar resultaat; `blockedTaskId`/samenvatting geeft aan welke |
| `succeeded` | Met succes voltooid                                                        |
| `failed`    | Voltooid met een fout                                                      |
| `cancelled` | Annulering aangevraagd en alle onderliggende taken afgehandeld              |
| `lost`      | De flow heeft zijn gezaghebbende onderliggende status verloren             |

## Duurzame statusgegevens en revisiebeheer

Flowregistraties blijven samen met taakregistraties bewaard in de gedeelde SQLite-statusdatabase (`~/.openclaw/state/openclaw.sqlite`, tabel `flow_runs`), zodat voortgang behouden blijft na herstarts van de Gateway. Elke schrijfbewerking verhoogt de `revision` van de flow; gelijktijdige schrijvers die een verouderde verwachte revisie doorgeven, krijgen een conflict en moeten de gegevens opnieuw lezen. De groei van het WAL wordt begrensd door automatische SQLite-checkpoints en periodieke passieve checkpoints, met afkappende checkpoints bij afsluiten. Het verouderde `flows/registry.sqlite`-zijbestand van oudere installaties wordt geïmporteerd door `openclaw doctor`.

## Annuleringsgedrag

`openclaw tasks flow cancel` stelt een blijvende annuleringsintentie in voor de flow, annuleert de actieve onderliggende taken en weigert nieuwe beheerde onderliggende taken. Zodra geen enkele onderliggende taak meer actief is, krijgt de flow de eindstatus `cancelled` — onmiddellijk, of via de onderhoudsscan als het langer duurt voordat onderliggende taken zijn afgehandeld. De intentie wordt bewaard, zodat een geannuleerde flow geannuleerd blijft, zelfs als de Gateway opnieuw wordt gestart voordat alle onderliggende taken zijn beëindigd.

## CLI-opdrachten

```bash
# Actieve en recente flows weergeven
openclaw tasks flow list [--status <status>] [--json]

# Details van een specifieke flow weergeven
openclaw tasks flow show <lookup> [--json]

# Een actieve flow en de bijbehorende actieve taken annuleren
openclaw tasks flow cancel <lookup>
```

| Opdracht                           | Beschrijving                                                            |
| --------------------------------- | ----------------------------------------------------------------------- |
| `openclaw tasks flow list`        | Gevolgde flows met synchronisatiemodus, status, revisie, controller en aantallen taken |
| `openclaw tasks flow show <id>`   | Eén flow inspecteren op flow-id of eigenaarsleutel, inclusief gekoppelde taken |
| `openclaw tasks flow cancel <id>` | Een actieve flow en de bijbehorende actieve taken annuleren             |

Flows vallen ook onder `openclaw tasks audit` (bevindingen voor verouderde of defecte flows) en `openclaw tasks maintenance` (voltooit vastgelopen annuleringen en verwijdert flows met een eindstatus na 7 dagen).

## Patroon voor betrouwbare geplande workflows

Behandel voor terugkerende workflows, zoals briefings over marktinformatie, de planning, orkestratie en betrouwbaarheidscontroles als afzonderlijke lagen:

1. Gebruik [Geplande taken](/nl/automation/cron-jobs) voor timing.
2. Gebruik een permanente Cron-sessie wanneer de workflow moet voortbouwen op eerdere context.
3. Gebruik [Lobster](/nl/tools/lobster) voor deterministische stappen, goedkeuringspoorten en hervattingstokens.
4. Gebruik Task Flow om de uitvoering met meerdere stappen te volgen over onderliggende taken, wachttijden, nieuwe pogingen en herstarts van de Gateway heen.

Voorbeeld van een Cron-configuratie:

```bash
openclaw cron add \
  --name "Marktinformatiebriefing" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Voer de Lobster-workflow voor marktinformatie uit. Controleer voordat je samenvat of de bronnen actueel zijn." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Gebruik `--session session:<id>` in plaats van `isolated` wanneer de terugkerende workflow een doelbewuste geschiedenis, samenvattingen van eerdere uitvoeringen of vaste context nodig heeft. Gebruik `isolated` wanneer elke uitvoering opnieuw moet beginnen en alle vereiste statusgegevens expliciet in de workflow zijn opgenomen.

Plaats binnen de workflow de betrouwbaarheidscontroles vóór de samenvattingsstap van het LLM:

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

Aanbevolen controles vooraf:

- Beschikbaarheid van de browser en profielkeuze, bijvoorbeeld `openclaw` voor beheerde statusgegevens of `user` wanneer een aangemelde Chrome-sessie vereist is. Zie [Browser](/nl/tools/browser).
- API-referenties en quota voor elke bron.
- Netwerkbereikbaarheid voor vereiste eindpunten.
- Vereiste hulpmiddelen die voor de agent zijn ingeschakeld, zoals `lobster`, `browser` en `llm-task`.
- Een foutbestemming die voor Cron is geconfigureerd, zodat mislukte controles vooraf zichtbaar zijn. Zie [Geplande taken](/nl/automation/cron-jobs#delivery-and-output).

Aanbevolen velden voor gegevensherkomst van elk verzameld item:

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Voorbeeldrapport",
  "content": "..."
}
```

Laat de workflow verouderde items vóór de samenvatting weigeren of als verouderd markeren. De LLM-stap mag alleen gestructureerde JSON ontvangen en moet de opdracht krijgen om `sourceUrl`, `retrievedAt` en `asOf` in de uitvoer te behouden. Gebruik [LLM-taak](/nl/tools/llm-task) wanneer je binnen de workflow een modelstap met schemavalidatie nodig hebt.

Verpak voor herbruikbare team- of communityworkflows de CLI, `.lobster`-bestanden en eventuele installatie-instructies als een skill of plugin en publiceer deze via [ClawHub](/clawhub). Bewaar workflowspecifieke waarborgen in dat pakket, tenzij de plugin-API een benodigde generieke mogelijkheid mist.

## Relatie tussen flows en taken

Flows coördineren taken, maar vervangen ze niet. Eén flow kan gedurende zijn levensduur meerdere achtergrondtaken aansturen. Gebruik `openclaw tasks` om afzonderlijke taakregistraties te inspecteren en `openclaw tasks flow` om de orkestrerende flow te inspecteren.

## Gerelateerd

- [Achtergrondtaken](/nl/automation/tasks) — het register voor losgekoppeld werk dat door flows wordt gecoördineerd
- [CLI: taken](/nl/cli/tasks) — CLI-opdrachtenreferentie voor `openclaw tasks flow`
- [Overzicht van automatisering](/nl/automation) — alle automatiseringsmechanismen in één oogopslag
- [Cron-taken](/nl/automation/cron-jobs) — geplande taken die flows van invoer kunnen voorzien
