---
read_when:
    - Je wilt de actuele OpenClaw-documentatie doorzoeken vanuit de terminal
    - Je moet weten welke gehoste zoek-API de docs-CLI aanroept
summary: CLI-referentie voor `openclaw docs` (doorzoek de actuele documentatie-index)
title: Documentatie
x-i18n:
    generated_at: "2026-07-27T06:08:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b0b575f0b76d40a53dd4f79c55fd65969a24eae27e27bd1c46d395f61fe89e42
    source_path: cli/docs.md
    workflow: 16
---

# `openclaw docs`

Doorzoek de live OpenClaw-documentatie-index vanuit de terminal.

## Gebruik

```bash
openclaw docs                       # toon het beginpunt van de documentatie en een voorbeeldzoekopdracht
openclaw docs <query...>            # doorzoek de live documentatie-index
```

| Argument     | Beschrijving                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | Vrije zoekopdracht. Zoekopdrachten met meerdere woorden worden met spaties samengevoegd en als één zoekopdracht verzonden. |

Zonder zoekopdracht toont `openclaw docs` de URL van het beginpunt van de documentatie en een voorbeeldzoekopdracht in plaats van een zoekopdracht uit te voeren.

## Voorbeelden

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## Werking

`openclaw docs` roept `https://docs.openclaw.ai/api/search` aan en geeft de JSON-resultaten weer. Voor de zoekaanvraag geldt een vaste time-out van 30 seconden.

## Uitvoer

In een uitgebreide (TTY-)terminal worden resultaten weergegeven als een kop gevolgd door een lijst met opsommingstekens: paginatitel, gekoppelde documentatie-URL en een kort fragment op de volgende regel. Bij lege resultaten wordt "Geen resultaten." weergegeven.

In niet-uitgebreide uitvoer (via een pipe, `--no-color`, scripts) worden dezelfde gegevens als Markdown weergegeven:

```markdown
# Documentatie doorzoeken: <query>

- [Titel](https://docs.openclaw.ai/...) - fragment
- [Titel](https://docs.openclaw.ai/...) - fragment
```

## Afsluitcodes

| Code | Betekenis                                                                  |
| ---- | ------------------------------------------------------------------------ |
| `0`  | De zoekopdracht is geslaagd, ook als er geen resultaten zijn.                       |
| `1`  | De API-aanroep voor het doorzoeken van de gehoste documentatie is mislukt; stderr toont de foutmelding. |

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Live documentatie](https://docs.openclaw.ai)
