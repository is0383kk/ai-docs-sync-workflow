---
read_when:
    - Je voert een upgrade uit van een configuratie die afgeleide commitments gebruikte
    - Je wilt eerder opgeslagen follow-uprecords bekijken of verwijderen
sidebarTitle: Commitments
summary: Status- en opruimrichtlijnen voor beëindigde afgeleide vervolgtoezeggingen
title: Afgeleide toezeggingen
x-i18n:
    generated_at: "2026-07-27T05:07:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

Het experiment met afgeleide toezeggingen is beëindigd. OpenClaw extraheert niet langer nieuwe
vervolgacties uit gesprekken en levert deze niet meer via Heartbeat, en het voormalige
configuratieblok `commitments` wordt verwijderd door `openclaw doctor --fix`.

Exacte herinneringen en geplande werkzaamheden blijven gebruikmaken van
[geplande taken](/nl/automation/cron-jobs). Duurzame gespreksfeiten horen thuis in
[geheugen](/nl/concepts/memory).

## Bestaande records

Eerder opgeslagen toezeggingen blijven in de gedeelde SQLite-statusdatabase staan, zodat een
upgrade de voor operators zichtbare geschiedenis niet vernietigt. Gebruik de verouderde onderhouds-CLI
om die rijen te bekijken of af te wijzen:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

Zie [`openclaw commitments`](/nl/cli/commitments) voor de referentie van de
onderhoudsopdracht.

## Gerelateerd

- [Geplande taken](/nl/automation/cron-jobs)
- [Geheugenoverzicht](/nl/concepts/memory)
- [Heartbeat](/nl/gateway/heartbeat)
