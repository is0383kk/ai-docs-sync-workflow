---
read_when:
    - Je moet inloggen op websites voor browserautomatisering
    - Je wilt updates op X/Twitter plaatsen
summary: Handmatig inloggen voor browserautomatisering en posten op X/Twitter
title: Browseraanmelding
x-i18n:
    generated_at: "2026-07-27T05:17:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bccd363cf7c9611f4687d50a92f7fb3e2fd1c1d67bb27a80c892f7ac58ae1f8f
    source_path: tools/browser-login.md
    workflow: 16
---

## Handmatig inloggen (aanbevolen)

Wanneer een site vereist dat je inlogt, log je handmatig in via het `openclaw`-profiel
van de hostbrowser. Geef je inloggegevens niet aan het model: geautomatiseerd inloggen
activeert vaak antibotmaatregelen en kan het account vergrendelen.

Gebruik de hostbrowser (handmatig ingelogd) voor zowel lezen (zoekopdrachten/threads) als
publiceren op X/Twitter en andere sites die gevoelig zijn voor bots. Browsersessies in een
sandbox activeren eerder botdetectie.

Terug naar de hoofddocumentatie voor de browser: [Browser](/nl/tools/browser).

## Welk Chrome-profiel wordt gebruikt?

OpenClaw beheert een speciaal Chrome-profiel met de naam `openclaw` (oranje getinte
interface), los van je dagelijkse browserprofiel.

Voor aanroepen van de browsertool door de agent:

- Standaardkeuze: de agent gebruikt zijn geïsoleerde `openclaw`-browser.
- Gebruik `profile="user"` alleen wanneer bestaande ingelogde sessies van belang zijn en je
  achter de computer zit om op een eventuele koppelingsprompt te klikken of deze goed te keuren.
- Als je meerdere gebruikersbrowserprofielen hebt, geef je het profiel expliciet op
  in plaats van te gokken.

Er zijn twee manieren om toegang te krijgen tot het `openclaw`-profiel:

1. Vraag de agent om de browser te openen en log vervolgens zelf in.
2. Open het via de CLI:

```bash
openclaw browser start
openclaw browser open https://x.com
```

Voor een niet-standaardprofiel plaats je `--browser-profile <name>` vóór de
subopdracht (standaard is `openclaw`):

```bash
openclaw browser --browser-profile <name> open https://x.com
```

## Sandboxing: toegang tot de hostbrowser toestaan

Als de agent in een sandbox wordt uitgevoerd, zijn de aanroepen van de `browser`-tool standaard gericht op de
browser in de sandbox, niet op de hostbrowser. Zo laat je de agent in plaats daarvan de hostbrowser gebruiken:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true,
        },
      },
    },
  },
}
```

CLI-aanroepen zijn altijd gericht op de hostbrowser en nooit op de sandbox. Je kunt de
hostbrowser dus zelf openen, ongeacht deze instelling:

```bash
openclaw browser --browser-profile openclaw open https://x.com
```

Zodra `sandbox.browser.allowHostControl: true` is ingesteld, kunnen de aanroepen van de `browser`-tool door de agent
ook op de host worden gericht. Je kunt ook sandboxing uitschakelen voor de
agent die updates publiceert.

## Gerelateerd

- [Browser](/nl/tools/browser)
- [Probleemoplossing voor Browser op Linux](/nl/tools/browser-linux-troubleshooting)
- [Probleemoplossing voor Browser met WSL2](/nl/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
