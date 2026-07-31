---
read_when:
    - Je wilt persistent geheugen dat in verschillende sessies en kanalen werkt
    - Je wilt door AI ondersteunde herinnering en gebruikersmodellering
summary: AI-native sessieoverstijgend geheugen via de Honcho-plugin
title: Honcho-geheugen
x-i18n:
    generated_at: "2026-07-27T05:07:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fadcf6d8e2505ab4fe6a81340695b7c8fee49c3cb4889665af13389941619117
    source_path: concepts/memory-honcho.md
    workflow: 16
---

[Honcho](https://honcho.dev) voegt AI-native geheugen toe aan OpenClaw via een
externe plugin. Gesprekken worden opgeslagen in een speciale service en in de
loop van de tijd worden gebruikers- en agentmodellen opgebouwd, waardoor je agent
sessieoverschrijdende context krijgt die verder gaat dan Markdown-bestanden in de werkruimte.

## Wat het biedt

- **Sessieoverschrijdend geheugen** - gesprekken blijven na elke beurt bewaard, zodat
  context behouden blijft na het opnieuw instellen van sessies, Compaction en het wisselen van kanaal.
- **Gebruikersmodellering** - Honcho onderhoudt een profiel voor elke gebruiker (voorkeuren,
  feiten, communicatiestijl) en voor de agent (persoonlijkheid, aangeleerd
  gedrag).
- **Semantisch zoeken** - zoek in observaties uit eerdere gesprekken, niet
  alleen in de huidige sessie.
- **Bewustzijn van meerdere agents** - bovenliggende agents volgen automatisch gestarte
  subagents, waarbij bovenliggende agents als waarnemers aan onderliggende sessies worden toegevoegd.

## Beschikbare tools

Honcho registreert tools die de agent tijdens gesprekken kan gebruiken:

**Gegevens ophalen (snel, geen LLM-aanroep):**

| Tool                        | Wat deze doet                                           |
| --------------------------- | ------------------------------------------------------ |
| `honcho_context`            | Volledige gebruikersrepresentatie over sessies heen               |
| `honcho_search_conclusions` | Semantisch zoeken in opgeslagen conclusies                |
| `honcho_search_messages`    | Berichten in sessies vinden (filteren op afzender, datum) |
| `honcho_session`            | Geschiedenis en samenvatting van de huidige sessie                    |

**Vragen en antwoorden (aangedreven door een LLM):**

| Tool         | Wat deze doet                                                              |
| ------------ | ------------------------------------------------------------------------- |
| `honcho_ask` | Vragen stellen over de gebruiker. `depth='quick'` voor feiten, `'thorough'` voor synthese |

## Aan de slag

Installeer de plugin en voer de configuratie uit:

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

De configuratieopdracht vraagt om je API-referenties, schrijft de configuratie en
migreert optioneel bestaande geheugenbestanden uit de werkruimte.

<Info>
Honcho kan volledig lokaal (zelfgehost) of via de beheerde API op
`api.honcho.dev` worden uitgevoerd. Voor de zelfgehoste optie zijn geen externe
afhankelijkheden vereist.
</Info>

## Configuratie

Instellingen staan onder `plugins.entries["openclaw-honcho"].config`:

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // omit for self-hosted
          workspaceId: "openclaw", // memory isolation
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

Laat voor zelfgehoste instanties `baseUrl` naar je lokale server verwijzen (bijvoorbeeld
`http://localhost:8000`) en laat de API-sleutel weg.

## Bestaand geheugen migreren

Als je bestaande geheugenbestanden in de werkruimte hebt (`USER.md`, `MEMORY.md`,
`IDENTITY.md`, `memory/`, `canvas/`), detecteert `openclaw honcho setup` deze en
biedt aan ze te migreren.

<Info>
Migratie is niet-destructief: bestanden worden naar Honcho geüpload. Originelen worden
nooit verwijderd of verplaatst.
</Info>

## Hoe het werkt

Na elke AI-beurt wordt het gesprek opgeslagen in Honcho. Zowel berichten van de gebruiker als
van de agent worden geobserveerd, zodat Honcho zijn modellen in de loop van de
tijd kan opbouwen en verfijnen.

Tijdens gesprekken vragen Honcho-tools de service op via de pluginhook
`before_prompt_build` van OpenClaw, waarbij relevante context wordt ingevoegd voordat het model
de prompt ziet.

## Honcho versus ingebouwd geheugen

|                   | Ingebouwd / QMD                | Honcho                              |
| ----------------- | ---------------------------- | ----------------------------------- |
| **Opslag**       | Markdown-bestanden in de werkruimte     | Speciale service (lokaal of gehost) |
| **Sessieoverschrijdend** | Via geheugenbestanden             | Automatisch, ingebouwd                 |
| **Gebruikersmodellering** | Handmatig (schrijven naar MEMORY.md)  | Automatische profielen                  |
| **Zoeken**        | Vector + trefwoord (hybride)    | Semantisch in observaties          |
| **Meerdere agents**   | Niet bijgehouden                  | Bewustzijn van bovenliggende/onderliggende agents              |
| **Afhankelijkheden**  | Geen (ingebouwd) of QMD-binair bestand | Installatie van plugin                      |

Honcho en het ingebouwde geheugensysteem kunnen samenwerken. Wanneer QMD is
geconfigureerd, komen aanvullende tools beschikbaar om lokale Markdown-bestanden
te doorzoeken naast het sessieoverschrijdende geheugen van Honcho.

## CLI-opdrachten

```bash
openclaw honcho setup                        # Configure API key and migrate files
openclaw honcho status                       # Check connection status
openclaw honcho ask <question>               # Query Honcho about the user
openclaw honcho search <query> [-k N] [-d D] # Semantic search over memory
```

## Verder lezen

- [Broncode van de plugin](https://github.com/plastic-labs/openclaw-honcho)
- [Honcho-documentatie](https://docs.honcho.dev)
- [Integratiehandleiding voor Honcho en OpenClaw](https://docs.honcho.dev/v3/guides/integrations/openclaw)

## Gerelateerd

- [Overzicht van geheugen](/nl/concepts/memory)
- [Ingebouwde geheugenengine](/nl/concepts/memory-builtin)
- [QMD-geheugenengine](/nl/concepts/memory-qmd)
- [Contextengines](/nl/concepts/context-engine)
