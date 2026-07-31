---
read_when:
    - Je wilt afgeleide toezeggingen voor vervolgacties bekijken
    - Je wilt openstaande check-ins negeren
    - Je controleert wat Heartbeat mogelijk aflevert
summary: CLI-referentie voor `openclaw commitments` (afgeleide vervolgacties inspecteren en negeren)
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-27T05:04:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

Inspecteer en verwijder records die zijn achtergelaten door het beëindigde experiment met afgeleide toezeggingen.
OpenClaw maakt of levert geen nieuwe toezeggingen meer, maar behoudt de onderhoudsopdracht
zodat upgrades bestaande SQLite-rijen kunnen controleren en opschonen.

Zonder subopdracht vermeldt `openclaw commitments` de openstaande toezeggingen.

## Gebruik

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## Opties

- `--all`: toon alle statussen in plaats van alleen openstaande toezeggingen.
- `--agent <id>`: filter op één agent-id.
- `--status <status>`: filter op status. Waarden: `pending`, `sent`,
  `dismissed`, `snoozed` of `expired`. Bij onbekende waarden wordt het programma met een fout afgesloten.
- `--json`: voer machineleesbare JSON uit.

`dismiss` markeert de opgegeven toezeggings-id's als `dismissed`.

## Voorbeelden

Openstaande toezeggingen weergeven:

```bash
openclaw commitments
```

Alle opgeslagen toezeggingen weergeven:

```bash
openclaw commitments --all
```

Filteren op één agent:

```bash
openclaw commitments --agent main
```

Uitgestelde toezeggingen zoeken:

```bash
openclaw commitments --status snoozed
```

Een of meer toezeggingen verwijderen:

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

Exporteren als JSON:

```bash
openclaw commitments --all --json
```

## Uitvoer

Tekstuitvoer toont het aantal toezeggingen, het pad naar de gedeelde SQLite-database, eventuele actieve filters
en één rij per toezegging:

- toezeggings-id
- status
- soort (`event_check_in`, `deadline_check`, `care_check_in` of `open_loop`)
- vroegste vervaltijd
- bereik (agent/kanaal/doel)
- voorgestelde tekst voor de check-in

JSON-uitvoer bevat het aantal, de actieve status- en agentfilters, het
pad naar de gedeelde SQLite-database en de volledige opgeslagen records.

## Gerelateerd

- [Afgeleide toezeggingen](/nl/concepts/commitments)
- [Overzicht van Memory](/nl/concepts/memory)
- [Heartbeat](/nl/gateway/heartbeat)
- [Geplande taken](/nl/automation/cron-jobs)
