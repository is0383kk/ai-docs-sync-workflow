---
read_when:
    - Je wilt semantisch geheugen indexeren of doorzoeken
    - Je lost problemen op met de beschikbaarheid of indexering van geheugen
    - Je wilt herinneringen uit het kortetermijngeheugen promoveren naar `MEMORY.md`
summary: CLI-referentie voor `openclaw memory` (status/index/search/promote/promote-explain/rem-harness/rem-backfill)
title: Geheugen
x-i18n:
    generated_at: "2026-07-27T05:28:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

Beheer semantische geheugenindexering, zoeken en promotie naar `MEMORY.md`.
Geleverd door de gebundelde Plugin `memory-core`, beschikbaar wanneer
`plugins.slots.memory` `memory-core` selecteert (de standaardinstelling). Andere geheugenplugins
bieden hun eigen CLI-naamruimten.

Gerelateerd: het concept [Geheugen](/nl/concepts/memory), [Dreaming](/nl/concepts/dreaming),
[Referentie voor geheugenconfiguratie](/nl/reference/memory-config), [Geheugenwiki](/nl/plugins/memory-wiki),
[wiki](/nl/cli/wiki), [Plugins](/nl/tools/plugin).

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

Zonder `--agent` wordt de opdracht uitgevoerd voor elke agent in `agents.entries`; als er geen agentenlijst is
geconfigureerd, wordt teruggevallen op de standaardagent.

| Vlag        | Effect                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | Controleer of de vectoropslag, embeddingprovider en semantische zoekfunctie gereed zijn (dit brengt extra provideraanroepen met zich mee). Gewone `memory status` blijft snel en slaat dit over; een onbekende vector-/semantische status betekent dat deze niet is gecontroleerd. Lexicale QMD-`searchMode: "search"` slaat semantische vectorcontroles altijd over, zelfs met `--deep`. |
| `--index`   | Indexeer opnieuw als de opslag vuil is. Impliceert `--deep`.                                                                                                                                                                                                                                                          |
| `--fix`     | Herstel verouderde recall-vergrendelingen en normaliseer promotiemetadata.                                                                                                                                                                                                                                               |
| `--json`    | Druk JSON af.                                                                                                                                                                                                                                                                                               |
| `--verbose` | Geef gedetailleerde logboeken per fase weer.                                                                                                                                                                                                                                                                             |

Als de regel `Dreaming` zelfs met `dreaming.enabled: true` op `off` blijft staan, of
geplande sweeps nooit lijken te worden uitgevoerd, is de beheerde Dreaming-Cron afhankelijk van het
activeren van de Heartbeat van de standaardagent om reconciliatie te starten. Zie
[Dreaming](/nl/concepts/dreaming) voor planningsdetails.

De status vermeldt ook eventuele extra zoekpaden uit `memory.search.extraPaths`.

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

Dezelfde afbakening per agent als `status`. `--force` voert een volledige herindexering uit in plaats van
een incrementele. `--verbose` toont per agent de provider, het model, de bronnen en
details over extra paden voordat de voortgang van het indexeren wordt weergegeven.

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- Zoekopdracht: positionele `[query]` of `--query <text>`. Als beide zijn ingesteld, heeft `--query`
  voorrang. Als geen van beide is ingesteld, geeft de opdracht een fout.
- `--agent <id>`: gebruikt standaard de standaardagent (niet de volledige agentenlijst).
- `--max-results <n>`: beperk het aantal resultaten (positief geheel getal).
- `--min-score <n>`: filter overeenkomsten onder deze score weg.

## `memory promote`

Rangschik kortetermijnkandidaten uit `memory/YYYY-MM-DD.md` en voeg desgewenst
de beste vermeldingen toe aan `MEMORY.md`.

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| Vlag                       | Standaard      | Effect                                                            |
| -------------------------- | ------------ | ----------------------------------------------------------------- |
| `--limit <n>`              |              | Maximumaantal kandidaten om te retourneren/toe te passen.                                   |
| `--min-score <n>`          | `0.75`       | Minimale gewogen promotiescore.                                 |
| `--min-recall-count <n>`   | `3`          | Minimaal vereist aantal recalls.                                    |
| `--min-unique-queries <n>` | `2`          | Minimaal vereist aantal verschillende zoekopdrachten.                            |
| `--apply`                  | alleen voorbeeld | Voeg geselecteerde kandidaten toe aan `MEMORY.md` en markeer ze als gepromoveerd. |
| `--include-promoted`       |              | Neem kandidaten op die al in eerdere cycli zijn gepromoveerd.           |
| `--json`                   |              | Druk JSON af.                                                       |

Deze CLI-standaardwaarden verschillen van de drempelwaarden voor de diepe fase van de geplande Dreaming-sweep
(zie [Dreaming](#dreaming) hieronder); geef expliciete vlaggen door om
het sweepgedrag te evenaren voor een eenmalige handmatige uitvoering.

Rangschikkingssignalen: recallfrequentie, relevantie van opgehaalde gegevens, diversiteit van zoekopdrachten,
temporele recentheid, consolidatie over meerdere dagen en rijkdom van afgeleide concepten, verkregen
uit zowel geheugenrecalls als dagelijkse opnamepassages, plus een lichte versterkingsbonus voor de lichte/REM-fase
bij herhaalde Dreaming-herbezoeken. Vóór het schrijven leest de promotie
de actuele dagelijkse notitie opnieuw, zodat wijzigingen of verwijderingen van kortetermijnfragmenten
sinds de rangschikking worden gerespecteerd in plaats van uit een verouderde momentopname te promoveren.

## `memory promote-explain`

Leg de opbouw van de score van één promotiekandidaat uit.

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>` komt overeen met de sleutel (exact of als subtekenreeks), het pad of de fragmenttekst
van een kandidaat.

## `memory rem-harness`

Bekijk REM-reflecties, kandidaatwaarheden en promotie-uitvoer van de diepe fase
zonder iets te schrijven.

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`: vul de harness vanuit historische dagelijkse bestanden van `YYYY-MM-DD.md`
  in plaats van de actuele werkruimte.
- `--grounded`: geef ook een onderbouwd voorbeeld van `What Happened` / `Reflections` /
  `Possible Lasting Updates` weer op basis van de historische notities.

## `memory rem-backfill`

Schrijf onderbouwde historische REM-samenvattingen naar `DREAMS.md` voor beoordeling in de gebruikersinterface.
Omkeerbaar.

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`: vereist tenzij `--rollback`/`--rollback-short-term`
  is ingesteld. Historische dagelijkse geheugenbestanden of map waaruit gegevens worden aangevuld.
- `--stage-short-term`: vul ook onderbouwde duurzame kandidaten in de actuele
  kortetermijnpromotieopslag in, zodat de normale diepe fase ze kan rangschikken.
- `--rollback`: verwijder eerder geschreven onderbouwde dagboekvermeldingen uit
  `DREAMS.md`.
- `--rollback-short-term`: verwijder eerder klaargezette onderbouwde kortetermijnkandidaten.

## Dreaming

Dreaming is het systeem voor geheugenconsolidatie op de achtergrond met drie samenwerkende
fasen, die in volgorde volgens één planning worden uitgevoerd: **licht** (kortetermijnmateriaal
sorteren/klaarzetten), **REM** (reflecteren en thema's naar voren brengen), **diep** (duurzame
feiten promoveren naar `MEMORY.md`). Alleen de diepe fase schrijft naar `MEMORY.md`.

- Schakel in met `plugins.entries.memory-core.config.dreaming.enabled: true`
  (standaard `false`); `memory-core` beheert de Cron-taak voor de sweep automatisch, geen handmatige
  `openclaw cron add` vereist.
- Schakel vanuit de chat met `/dreaming on|off`; inspecteer met `/dreaming status`
  (of `/dreaming`/`/dreaming help`). `on`/`off` vereist de status van kanaaleigenaar
  of Gateway-`operator.admin`; `status` en hulp blijven beschikbaar voor iedereen die
  de opdracht kan aanroepen.
- Menselijk leesbare fase-uitvoer gaat naar `DREAMS.md` (of een bestaande `dreams.md`).
  Standaard (`dreaming.storage.mode: "separate"`) schrijft elke fase ook een
  zelfstandig rapport naar `memory/dreaming/<phase>/YYYY-MM-DD.md`; stel `mode:
"inline"` in om rapporten in plaats daarvan in het dagelijkse geheugenbestand op te nemen, of `"both"`
  voor beide.
- Geplande en handmatige uitvoeringen van `memory promote` gebruiken dezelfde rangschikkingssignalen
  voor de diepe fase; alleen de standaarddrempelwaarden verschillen (zie de tabel hierboven tegenover
  de geplande standaardwaarden hieronder).
- Geplande uitvoeringen worden verdeeld over de geheugenwerkruimte van elke geconfigureerde agent.

Geplande standaardwaarden (`plugins.entries.memory-core.config.dreaming`):

| Sleutel                                    | Standaard     |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

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

Volledige lijst met sleutels en fasedetails: [Dreaming](/nl/concepts/dreaming),
[Referentie voor geheugenconfiguratie](/nl/reference/memory-config#dreaming).

## SecretRef-afhankelijkheid van de Gateway

Als externe API-sleutelvelden voor Active Memory zijn geconfigureerd als SecretRefs, lossen `memory`-opdrachten
deze op vanuit de actieve Gateway-momentopname; als de Gateway
niet beschikbaar is, mislukt de opdracht onmiddellijk. Hiervoor is een Gateway vereist die de
methode `secrets.resolve` ondersteunt; oudere Gateways retourneren een fout voor een onbekende methode.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Geheugenoverzicht](/nl/concepts/memory)
