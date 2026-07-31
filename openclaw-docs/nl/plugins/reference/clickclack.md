---
read_when:
    - Je installeert, configureert of controleert de clickclack-plugin
summary: Voegt het Clickclack-kanaal toe voor het verzenden en ontvangen van OpenClaw-berichten.
title: Clickclack-plugin
x-i18n:
    generated_at: "2026-07-27T06:27:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fcb39341009946dc38a12cc24496e65fd704ed3f2f9aff44bb2dd29fdedaef26
    source_path: plugins/reference/clickclack.md
    workflow: 16
---

# Clickclack-plugin

Voegt het Clickclack-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

## Distributie

- Pakket: `@openclaw/clickclack`
- Installatieroute: npm; ClawHub: `clawhub:@openclaw/clickclack`

## Oppervlak

kanalen: `clickclack`; contracten: `tools`

<!-- openclaw-plugin-reference:manual-start -->

De plugin kan optioneel voor elke OpenClaw-sessie een met de levenscyclus gesynchroniseerd ClickClack-kanaal maken. Beheerde discussiekanalen gebruiken een nevensessie van dezelfde agent voor observatie en doorsturen, terwijl de gekoppelde hoofdsessie een uitsluitend voor ophalen bestemde `discussion`-tool ontvangt. Zie [ClickClack-sessiediscussies](/nl/channels/clickclack#session-discussions)
voor de vereisten voor configuratie en zichtbaarheid van sessietools.

<!-- openclaw-plugin-reference:manual-end -->

## Gerelateerde documentatie

- [clickclack](/nl/channels/clickclack)
