---
read_when:
    - Je wilt records van achtergrondtaken inspecteren, controleren of annuleren
    - Je documenteert TaskFlow-opdrachten onder `openclaw tasks flow`
summary: CLI-referentie voor `openclaw tasks` (register van achtergrondtaken en TaskFlow-status)
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-27T06:10:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

Inspecteer duurzame achtergrondtaken en de TaskFlow-status. Zonder subopdracht is
`openclaw tasks` gelijkwaardig aan `openclaw tasks list`.

Zie [Achtergrondtaken](/nl/automation/tasks) voor het levenscyclus- en afleveringsmodel
en de sectie `tasks audit` voor volledige beschrijvingen van bevindingen.

## Gebruik

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## Hoofdopties

| Vlag               | Beschrijving                                                                                       |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | Voer JSON uit.                                                                                     |
| `--runtime <name>` | Filter op soort: `subagent`, `acp`, `cron` of `cli`.                                               |
| `--status <name>`  | Filter op status: `queued`, `running`, `succeeded`, `failed`, `timed_out`, `cancelled` of `lost`. |

## Subopdrachten

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

Toont bijgehouden achtergrondtaken met de nieuwste eerst.

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

Toont één taak op basis van taak-ID, uitvoerings-ID of sessiesleutel.

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

Wijzigt het meldingsbeleid voor een actieve taak.

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

Annuleert een actieve achtergrondtaak.

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

Brengt verouderde, verloren, niet-afgeleverde of anderszins inconsistente taak- en
TaskFlow-records aan het licht. Verloren taken die tot `cleanupAfter` worden bewaard, zijn waarschuwingen;
verlopen of niet-gemarkeerde verloren taken zijn fouten.

`--code` accepteert taakcodes (`stale_queued`, `stale_running`, `lost`,
`delivery_failed`, `missing_cleanup`, `inconsistent_timestamps`) en TaskFlow-codes
(`restore_failed`, `stale_waiting`, `stale_blocked`,
`cancel_stuck`, `missing_linked_tasks`, `blocked_task_missing`). Zie
[Achtergrondtaken](/nl/automation/tasks) voor details over de ernst en activeringsvoorwaarde per
code.

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

Toont een voorbeeld van of past afstemming, opschoningsmarkering en verwijdering
van taken en TaskFlow toe, evenals het opschonen van het sessieregister voor verouderde cron-uitvoeringen.

Voor cron-taken gebruikt de afstemming opgeslagen uitvoeringslogboeken en taakstatussen voordat
een oude actieve taak als `lost` wordt gemarkeerd, zodat voltooide cron-uitvoeringen geen
onterechte auditfouten worden alleen omdat de runtime-status in het geheugen van de Gateway verdwenen is.
Een offline CLI-audit is niet gezaghebbend voor de proceslokale verzameling actieve cron-taken
van de Gateway. CLI-taken met een uitvoerings-ID/bron-ID worden als `lost` gemarkeerd wanneer
hun actieve Gateway-uitvoeringscontext verdwenen is, zelfs als een oude onderliggende sessierij
blijft bestaan.

Bij toepassing verwijdert onderhoud ook `cron:<jobId>:run:<uuid>`-sessieregisterrijen
ouder dan 7 dagen, waarbij momenteel actieve cron-taken behouden blijven
en sessierijen die niet bij cron horen ongewijzigd blijven.

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

Inspecteert of annuleert duurzame TaskFlow-status onder het takenregister.
`flow list --status` accepteert `queued`, `running`, `waiting`, `blocked`,
`succeeded`, `failed`, `cancelled` of `lost`.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Achtergrondtaken](/nl/automation/tasks)
