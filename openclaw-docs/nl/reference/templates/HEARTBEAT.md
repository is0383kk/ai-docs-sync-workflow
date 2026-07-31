---
read_when:
    - Een werkruimte handmatig opzetten
summary: Werkruimtesjabloon voor HEARTBEAT.md
title: HEARTBEAT.md-sjabloon
x-i18n:
    generated_at: "2026-07-27T06:09:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md-sjabloon

`HEARTBEAT.md` bevindt zich in de agentwerkruimte en bevat de periodieke Heartbeat-checklist. Houd het leeg, of laat het alleen witruimte, Markdown-opmerkingen, ATX-koppen, lege lijststubs (`- `, `* [ ]`) of fence-markeringen bevatten, zodat OpenClaw de aanroep van het Heartbeat-model volledig overslaat (`reason=empty-heartbeat-file`).

Standaard meegeleverde inhoud:

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# Houd dit bestand leeg (of laat het alleen opmerkingen bevatten) om Heartbeat-API-aanroepen over te slaan.

# Voeg hieronder een korte checklist toe wanneer de Heartbeat gedeelde context moet controleren.
```

Voeg alleen een korte checklist onder de commentaarregels toe wanneer één Heartbeat-beurt de items gezamenlijk moet controleren. Houd deze klein: Heartbeat-uitvoeringen lezen dit bestand bij elke tick (standaard elke 30 minuten), dus opgeblazen instructies verbruiken bij elke activering tokens.

Maak voor onafhankelijk geplande controles of controles die alleen bij het verstrijken van de termijn worden uitgevoerd [Cron-taken](/nl/automation/cron-jobs). Heartbeat-kladruimte ondersteunt geen planningssyntaxis meer. Voer `openclaw doctor --fix` uit om oudere `tasks:`-blokken te converteren.

## Gerelateerd

- [Heartbeat](/nl/gateway/heartbeat)
- [Heartbeat-configuratie](/nl/gateway/config-agents)
