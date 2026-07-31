---
read_when:
    - Statusindicatoren van de Mac-app debuggen
summary: Hoe de macOS-app de status van de Gateway en kanalen rapporteert
title: Statuscontroles (macOS)
x-i18n:
    generated_at: "2026-07-27T05:05:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# Statuscontroles op macOS

Zo lees je de status van een gekoppeld kanaal in de menubalk-app.

## Menubalk

Statusstip:

- Groen: gekoppeld + controle in orde.
- Oranje: gekoppeld, maar een kanaalcontrole meldt een verslechterde status/niet verbonden.
- Rood: nog niet gekoppeld.

De secundaire regel toont "gekoppeld · authenticatie 12m" of geeft de reden voor de fout weer.
"Run Health Check Now" in het menu start een controle op aanvraag.

## Instellingen

- Het tabblad Algemeen toont een statuskaart: statusstip, samenvattingsregel (koppelingsstatus +
  leeftijd van authenticatie) en een optionele regel met foutdetails, met de knoppen **Nu opnieuw proberen** en
  **Logboeken openen**.
- Het **tabblad Kanalen** toont de status en bedieningselementen per kanaal (QR-code voor aanmelding,
  afmelden, controle, laatste verbroken verbinding/fout) voor WhatsApp en Telegram.

## Hoe de controle werkt

De app roept elke ~60s en op aanvraag de `health`-RPC van de Gateway aan via de bestaande WebSocket-
verbinding (dus niet via een CLI-shellopdracht). De RPC laadt
inloggegevens en rapporteert de status zonder berichten te verzenden. De app slaat de laatste
goede momentopname en de laatste fout afzonderlijk op, zodat de UI direct wordt geladen en
niet flikkert wanneer deze offline is.

## Bij twijfel

Gebruik de CLI-flow in [Gateway-status](/nl/gateway/health) (`openclaw status`,
`openclaw status --deep`, `openclaw health --json`) en voer
`openclaw logs --follow` uit, gefilterd op `web-heartbeat` / `web-reconnect`.

## Gerelateerd

- [Gateway-status](/nl/gateway/health)
- [macOS-app](/nl/platforms/macos)
