---
read_when:
    - Werken aan telemetrie-/privacycontroles
    - Vragen over welke gegevens worden verzameld
summary: Installatietelemetrie die door de ClawHub-CLI wordt verzameld en hoe je je hiervoor kunt afmelden.
x-i18n:
    generated_at: "2026-07-27T05:26:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# Telemetrie

ClawHub gebruikt minimale CLI-telemetrie om geaggregeerde aantallen installaties van Skills en Plugins te berekenen.

## Wanneer telemetrie wordt verzameld

Telemetrie wordt alleen verzonden wanneer:

- Je bent aangemeld bij de CLI.
- Je voert `clawhub install <slug>` uit of voltooit een geauthenticeerde installatie van
  `openclaw plugins install clawhub:<package>`.
- Telemetrie is **niet uitgeschakeld** (zie ‘Uitschakelen’ hieronder).

Als je niet bent aangemeld, wordt er niets gerapporteerd.

## Wat we verzamelen

Nadat een Skill of Plugin is geïnstalleerd en de lokale installatierecord is opgeslagen, verzendt de CLI
één installatiegebeurtenis op basis van beste inspanning.

De gebeurtenis bevat:

- De slug van de geïnstalleerde Skill of de canonieke pakketnaam van de Plugin.
- `version`: de geïnstalleerde versie, indien bekend.

### Wat we _niet_ verzamelen

- Geen mappaden of van mappen afgeleide identificatoren.
- Geen bestandsinhoud.
- Geen logboeken per uitvoering, prompts of andere CLI-uitvoer.

## Aantallen installaties

Voor Skills houdt ClawHub het volgende bij:

- `installsAllTime`: unieke gebruikers die ten minste één CLI-installatie van de Skill hebben gerapporteerd.
- `installsCurrent`: unieke gebruikers die een installatie hebben gerapporteerd en hun
  telemetrie niet hebben verwijderd.

Voor Plugins telt ClawHub de eerste geslaagde installatie die door elke gebruiker voor elk pakket wordt gerapporteerd.
Herhaalde installaties en updates vernieuwen de geregistreerde versie zonder het geaggregeerde
aantal installaties te verhogen.

## Transparantie en gebruikersinstellingen

Iedereen ziet alleen **geaggregeerde installatietellers**.

Als je jouw account verwijdert, worden ook je telemetriegegevens verwijderd en telt hun bijdrage niet meer mee voor de
installatietellers.

## Telemetrie uitschakelen

Stel de omgevingsvariabele in:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

Als deze is ingesteld, verzendt de CLI geen installatietelemetrie.
