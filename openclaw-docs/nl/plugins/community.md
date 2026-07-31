---
doc-schema-version: 1
read_when:
    - Je wilt OpenClaw-plugins van derden vinden
    - Je wilt je eigen plugin publiceren of vermelden op ClawHub
summary: Vind en publiceer door de community onderhouden OpenClaw-plugins
title: Communityplugins
x-i18n:
    generated_at: "2026-07-27T05:56:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

Communityplugins zijn pakketten van derden die OpenClaw uitbreiden met
kanalen, tools, providers, hooks of andere mogelijkheden. Gebruik
[ClawHub](/clawhub) als het primaire platform om openbare communityplugins
te vinden.

## Plugins vinden

Doorzoek ClawHub vanuit de CLI:

```bash
openclaw plugins search "calendar"
```

Installeer een ClawHub-plugin met een expliciet bronvoorvoegsel:

```bash
openclaw plugins install clawhub:<package-name>
```

npm blijft tijdens de overgang bij de lancering een ondersteund direct installatiepad:

```bash
openclaw plugins install npm:<package-name>
```

Gebruik [Plugins beheren](/nl/plugins/manage-plugins) voor veelvoorkomende voorbeelden
van installeren, bijwerken, inspecteren en verwijderen. Gebruik
[`openclaw plugins`](/nl/cli/plugins) voor de volledige opdrachtreferentie en
regels voor bronselectie.

## Plugins publiceren

Publiceer openbare communityplugins op ClawHub, zodat OpenClaw-gebruikers
ze kunnen vinden en installeren. ClawHub beheert de actuele pakketvermelding,
releasegeschiedenis, scanstatus en installatieaanwijzingen; de documentatie
houdt geen statische catalogus met plugins van derden bij.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Zorg vóór publicatie dat de plugin pakketmetadata, een pluginmanifest,
installatiedocumentatie en een duidelijke onderhoudseigenaar heeft. ClawHub
valideert het eigenaarsbereik, de pakketnaam, versie, bestandslimieten en
bronmetadata voordat een release wordt gemaakt. Nieuwe releases blijven
vervolgens verborgen op de normale installatie- en downloadplatforms totdat
de beoordeling en verificatie zijn afgerond.

Checklist vóór publicatie:

| Vereiste                 | Waarom                                               |
| ------------------------ | ---------------------------------------------------- |
| Gepubliceerd op ClawHub  | Gebruikers hebben `openclaw plugins install`-aanwijzingen nodig |
| Openbare GitHub-repository | Bronbeoordeling, issuetracking, transparantie      |
| Installatie- en gebruiksdocumentatie | Gebruikers moeten weten hoe ze de plugin configureren |
| Actief onderhoud         | Recente updates of responsieve afhandeling van issues |

Volledig publicatiecontract:

- [Publiceren op ClawHub](/nl/clawhub/publishing) - eigenaren, bereiken, releases,
  beoordeling, pakketvalidatie en pakketoverdracht
- [Plugins bouwen](/nl/plugins/building-plugins) - de structuur van het pluginpakket
  en de workflow voor de eerste publicatie
- [Pluginmanifest](/nl/plugins/manifest) - velden van het native pluginmanifest

## Gerelateerd

- [Plugins](/nl/tools/plugin) - installeren, configureren, opnieuw starten en problemen oplossen
- [Plugins beheren](/nl/plugins/manage-plugins) - opdrachtvoorbeelden
- [Publiceren op ClawHub](/nl/clawhub/publishing) - regels voor publicatie en releases
