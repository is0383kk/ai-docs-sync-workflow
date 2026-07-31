---
read_when:
    - Scripts uitvoeren vanuit de repo
    - Scripts onder ./scripts toevoegen of wijzigen
summary: 'Repositoriescripts: doel, reikwijdte en veiligheidsopmerkingen'
title: Scripts
x-i18n:
    generated_at: "2026-07-27T05:08:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` bevat hulpscripts voor lokale workflows en operationele taken. Gebruik deze wanneer een taak duidelijk aan een script is gekoppeld; geef anders de voorkeur aan de CLI.

## Conventies

- Scripts zijn **optioneel**, tenzij ernaar wordt verwezen in documentatie of releasechecklists.
- Geef de voorkeur aan CLI-interfaces wanneer die bestaan (voorbeeld: `openclaw models status --check`).
- Ga ervan uit dat scripts hostspecifiek zijn; lees ze voordat je ze op een nieuwe machine uitvoert.

## Scripts voor authenticatiebewaking

Algemene modelauthenticatie wordt behandeld in [Authenticatie](/nl/gateway/authentication). De onderstaande scripts vormen een afzonderlijk, optioneel systeem voor het bewaken van een **Claude Code CLI-abonnementstoken** op een externe/headless host en het opnieuw authenticeren vanaf een telefoon:

- `scripts/setup-auth-system.sh` - eenmalige installatie: controleert de huidige authenticatie, helpt een langlevende `claude setup-token` te genereren en toont installatiestappen voor systemd/Termux.
- `scripts/claude-auth-status.sh [full|json|simple]` - controleert de authenticatiestatus van Claude Code en OpenClaw.
- `scripts/auth-monitor.sh` - controleert periodiek de status en stuurt een melding (via OpenClaw send en/of ntfy.sh) wanneer de vervaldatum van het token nadert. Omgevingsvariabelen: `WARN_HOURS` (standaard `2`), `NOTIFY_PHONE`, `NOTIFY_NTFY`. Voer dit volgens een schema uit via de meegeleverde `scripts/systemd/openclaw-auth-monitor.{service,timer}` (elke 30 minuten).
- `scripts/mobile-reauth.sh` - voert `claude setup-token` opnieuw uit en toont URL's die je op een telefoon kunt openen, voor gebruik via SSH vanuit Termux.
- `scripts/termux-quick-auth.sh`, `scripts/termux-auth-widget.sh`, `scripts/termux-sync-widget.sh` - Termux:Widget-scripts die via SSH verbinding maken met de host, een statusmelding tonen en de console/instructies voor herauthenticatie openen wanneer de authenticatie is verlopen.

## GitHub-leeshulpmiddel

Gebruik `scripts/gh-read` wanneer je wilt dat `gh` een installatietoken van een GitHub App gebruikt voor repo-gebonden leesaanroepen, terwijl de normale `gh` voor schrijfacties aangemeld blijft met jouw persoonlijke account.

Vereiste omgevingsvariabelen:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Optionele omgevingsvariabelen:

- `OPENCLAW_GH_READ_INSTALLATION_ID` wanneer je het op de repo gebaseerde opzoeken van de installatie wilt overslaan
- `OPENCLAW_GH_READ_PERMISSIONS` als een door komma's gescheiden overschrijving voor de aan te vragen subset van leesrechten

Volgorde voor het bepalen van de repo:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Voorbeelden:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## Bij het toevoegen van scripts

- Houd scripts doelgericht en gedocumenteerd.
- Voeg een korte vermelding toe aan het relevante document (of maak er een als dit ontbreekt).

## Gerelateerd

- [Testen](/nl/help/testing)
- [Live testen](/nl/help/testing-live)
