---
read_when:
    - Je gebruikt DM's in de koppelingsmodus en moet afzenders goedkeuren
summary: CLI-referentie voor `openclaw pairing` (koppelingsverzoeken goedkeuren/weergeven)
title: Koppelen
x-i18n:
    generated_at: "2026-07-27T05:41:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

Keur DM-koppelingsverzoeken goed of inspecteer ze voor kanalen die koppeling ondersteunen (alleen chat-DM's - voor het koppelen van nodes/apparaten wordt `openclaw devices` gebruikt).

Gerelateerd: [Koppelingsflow](/nl/channels/pairing)

Dezelfde openstaande verzoeken kunnen in de Control UI worden beoordeeld onder **Settings →
Channels → DM access requests**. De Control UI ondersteunt goedkeuren, optionele
melding aan de aanvrager en afwijzen. Afwijzen verwijdert het huidige verzoek, maar
blokkeert de afzender niet permanent.

## Opdrachten

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

Geef openstaande koppelingsverzoeken voor één kanaal weer.

| Optie                   | Beschrijving                          |
| ----------------------- | ------------------------------------- |
| `[channel]`             | positionele kanaal-id                 |
| `--channel <channel>`   | expliciete kanaal-id                  |
| `--account <accountId>` | account-id voor kanalen met meerdere accounts |
| `--json`                | machineleesbare uitvoer               |

Als meerdere kanalen met koppelingsondersteuning zijn geconfigureerd, geef je een kanaal positioneel of met `--channel` door. Uitbreidingskanalen werken zolang de kanaal-id geldig is.

## `pairing approve`

Keur een openstaande koppelingscode goed en sta die afzender toe.

Gebruik:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` wanneer precies één kanaal met koppelingsondersteuning is geconfigureerd

Opties: `--channel <channel>`, `--account <accountId>`, `--notify` (stuur via hetzelfde kanaal een bevestiging terug naar de aanvrager).

### Initiële eigenaar instellen

Als `commands.ownerAllowFrom` leeg is wanneer je een koppelingscode goedkeurt, registreert de CLI de goedgekeurde afzender ook als opdrachteigenaar, met een kanaalspecifieke vermelding zoals `telegram:123456789`. Hiermee wordt alleen de eerste eigenaar ingesteld - latere goedkeuringen van koppelingen vervangen of verruimen `commands.ownerAllowFrom` nooit. De Control UI presenteert deze verhoging als een afzonderlijk, met `operator.admin` beveiligd selectievakje in plaats van deze automatisch toe te passen.

De opdrachteigenaar is het account van de menselijke beheerder dat opdrachten mag uitvoeren die uitsluitend voor de eigenaar bestemd zijn en gevaarlijke acties mag goedkeuren, zoals `/diagnostics`, `/export-session`, `/export-trajectory`, `/config` en goedkeuringen voor uitvoering. Door koppeling kan een afzender alleen met de agent communiceren; op zichzelf verleent dit geen eigenaarsrechten, behalve bij deze eenmalige initiële instelling.

Als je een afzender hebt goedgekeurd voordat deze initiële instelling bestond, voer je `openclaw doctor` uit; dit geeft een waarschuwing wanneer geen opdrachteigenaar is geconfigureerd en toont de exacte opdracht `openclaw config set commands.ownerAllowFrom ...` om dit te verhelpen.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Kanaalkoppeling](/nl/channels/pairing)
