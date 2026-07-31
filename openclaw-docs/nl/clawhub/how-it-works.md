---
read_when:
    - Inzicht in vermeldingen, versies, installaties, publicatie en moderatie
summary: Hoe ClawHub-vermeldingen, versies, installaties, publicaties, scans en updates werken.
x-i18n:
    generated_at: "2026-07-27T06:07:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 747079343899e42d00f84b00c553447abe0b83f2c4f1c9cdbf54725e34779eaf
    source_path: clawhub/how-it-works.md
    workflow: 16
---

# Hoe ClawHub werkt

ClawHub is de registerlaag voor OpenClaw-Skills en -plugins. Het biedt gebruikers
een plek om pakketten te ontdekken, uitgevers een plek om versies uit te brengen
en OpenClaw voldoende metadata om die pakketten veilig te installeren en bij te werken.

## Registerrecords

Elke openbare vermelding is een registerrecord met:

- een eigenaar en slug of pakketnaam
- een of meer gepubliceerde versies
- metadata, samenvatting, bestanden en bronvermelding
- informatie over de changelog en tags, zoals `latest`
- signalen voor downloads, installaties en sterren
- status van beveiligingsscans en moderatie

De vermeldingspagina is de canonieke plek waar gebruikers kunnen controleren wat
een Skill of plugin beweert te doen voordat ze deze installeren.

## Skills

Een Skill is een tekstbundel met versiebeheer die is opgebouwd rond `SKILL.md`.
Deze kan ondersteunende bestanden, voorbeelden, sjablonen en scripts bevatten.

ClawHub leest de frontmatter van `SKILL.md` om inzicht te krijgen in de naam,
beschrijving, vereisten, omgevingsvariabelen en metadata van de Skill. Nauwkeurige
metadata zijn belangrijk, omdat ze gebruikers helpen beslissen of ze de Skill willen
installeren en geautomatiseerde scans helpen verschillen tussen gedeclareerd en
waargenomen gedrag te detecteren.

Zie [Skill-indeling](/clawhub/skill-format).

## Plugins

Plugins zijn verpakte OpenClaw-uitbreidingen. ClawHub bewaart pakketmetadata,
compatibiliteitsinformatie, bronlinks, artefacten en versierecords.

Wanneer OpenClaw een plugin vanuit ClawHub installeert, controleert het vóór de
installatie de opgegeven compatibiliteitsmetadata. Pakketrecords kunnen
API-compatibiliteit, de minimale Gateway-versie, hostdoelen, omgevingsvereisten
en artefact-digests bevatten.

Gebruik een expliciete ClawHub-installatiebron als je wilt dat het register de
gezaghebbende bron is:

```bash
openclaw plugins install clawhub:<package>
```

## Publiceren

Bij publicatie wordt een nieuw, onveranderlijk versierecord gemaakt. Uitgevers
gebruiken de CLI `clawhub` voor geauthenticeerde registerworkflows:

```bash
clawhub skill publish ./my-skill
clawhub package publish <source> --family code-plugin --dry-run
clawhub package publish <source> --family code-plugin
```

Gebruik proefuitvoeringen om vóór het uploaden een voorbeeld van de vastgestelde
payload te bekijken. Openbare pagina's tonen vervolgens de gepubliceerde metadata,
bestanden, bronvermelding en scanstatus.

## Installaties en updates

OpenClaw-installatieopdrachten gebruiken ClawHub als pakketbron:

```bash
openclaw skills install @openclaw/demo
openclaw plugins install clawhub:<package>
```

OpenClaw registreert metadata over de installatiebron, zodat updates later
hetzelfde registerpakket kunnen vinden. De ClawHub-CLI ondersteunt ook workflows
voor het rechtstreeks installeren en bijwerken van Skills voor gebruikers die
door het register beheerde Skill-mappen buiten een volledige OpenClaw-werkruimte
willen gebruiken.

## Beveiligingsstatus

ClawHub staat open voor publicatie, maar releases zijn nog steeds onderworpen aan
uploadcontroles, geautomatiseerde controles, gebruikersmeldingen en maatregelen
van moderators.

Openbare pagina's tonen scansamenvattingen wanneer die beschikbaar zijn. Inhoud
die wordt vastgehouden, verborgen of geblokkeerd, kan uit openbare zoekresultaten
en installatieworkflows verdwijnen, terwijl deze voor diagnostiek zichtbaar blijft
voor de eigenaar.

Zie [Beveiliging](/clawhub/security), [Beveiligingsaudits](/clawhub/security-audits),
[Moderatie en accountveiligheid](/clawhub/moderation) en
[Aanvaardbaar gebruik](/clawhub/acceptable-usage).

## API-toegang

ClawHub biedt openbare lees-API's voor ontdekking, zoeken, pakketdetails en
downloads. Catalogi van derden mogen deze API's gebruiken wanneer ze naar de
canonieke ClawHub-vermelding verwijzen, snelheidslimieten respecteren en niet
de indruk wekken dat ze worden aanbevolen.

Zie [Openbare API](/clawhub/api) en [HTTP-API](/clawhub/http-api).
