---
read_when:
    - ClawHub voor het eerst gebruiken
    - Een skill of plugin uit het register installeren
    - Publiceren op ClawHub
summary: 'Ga aan de slag met ClawHub: zoek, installeer, werk Skills of plugins bij en publiceer ze.'
x-i18n:
    generated_at: "2026-07-27T06:07:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f6d61bd32a359a843e68140cc90b4ff4bcc64645ea425ea4654c668d6d3d04ec
    source_path: clawhub/quickstart.md
    workflow: 16
---

# Snelstart

ClawHub is een register voor OpenClaw-Skills en -plugins.

Gebruik OpenClaw wanneer je onderdelen in OpenClaw installeert. Gebruik de `clawhub`-CLI
wanneer je je aanmeldt, publiceert, je eigen vermeldingen beheert of
registerspecifieke workflows gebruikt.

## Een Skill zoeken en installeren

Zoeken vanuit OpenClaw:

```bash
openclaw skills search "calendar"
```

Een Skill installeren:

```bash
openclaw skills install @openclaw/demo
```

Geïnstalleerde Skills bijwerken:

```bash
openclaw skills update --all
```

OpenClaw registreert waar de Skill vandaan kwam, zodat latere updates deze
via ClawHub kunnen blijven vinden.

## Een Plugin zoeken en installeren

Zoeken vanuit OpenClaw:

```bash
openclaw plugins search "calendar"
```

Een door ClawHub gehoste Plugin installeren met een expliciete ClawHub-bron:

```bash
openclaw plugins install clawhub:<package>
```

Geïnstalleerde plugins bijwerken:

```bash
openclaw plugins update --all
```

Gebruik het voorvoegsel `clawhub:` wanneer je wilt dat OpenClaw het pakket via
ClawHub vindt in plaats van via npm of een andere bron.

## Aanmelden om te publiceren

Installeer de ClawHub-CLI:

```bash
npm i -g clawhub
# of
pnpm add -g clawhub
```

Meld je aan met GitHub:

```bash
clawhub login
clawhub whoami
```

Headless omgevingen kunnen een API-token uit de ClawHub-webinterface gebruiken:

```bash
clawhub login --token clh_...
```

## Een Skill publiceren

Een Skill is een map met een vereist bestand `SKILL.md` en optionele
ondersteunende bestanden.

```bash
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "My Skill" \
  --changelog "Initial release"
```

De opdracht slaat ongewijzigde inhoud over. Nieuwe Skills beginnen bij `1.0.0`; bij latere wijzigingen
wordt automatisch de volgende patchversie gepubliceerd. Gebruik `--dry-run` voor een voorbeeldweergave of
`--version` om een expliciete versie te kiezen.

Controleer vóór publicatie de metadata in `SKILL.md`. Declareer vereiste
omgevingsvariabelen, hulpmiddelen en machtigingen, zodat gebruikers vóór de
installatie begrijpen wat de Skill nodig heeft. Zie [Skill-indeling](/clawhub/skill-format).

Voor repository's die meerdere Skills bevatten, roept de herbruikbare GitHub-workflow
`skill publish` aan voor elke directe Skill-map onder `skills/`:

```yaml
jobs:
  preview:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      dry_run: true
```

## Een Plugin publiceren

Publiceer een Plugin vanuit een lokale map, een GitHub-repository, een GitHub-referentie of een
bestaand archief:

```bash
clawhub package publish <source> --family code-plugin --dry-run
clawhub package publish <source> --family code-plugin
```

Gebruik eerst `--dry-run` om een voorbeeld te bekijken van de gevonden pakketmetadata, compatibiliteitsvelden,
brontoeschrijving en het uploadplan, zonder te publiceren.

Codeplugins moeten in `package.json` compatibiliteitsmetadata voor OpenClaw bevatten,
waaronder `openclaw.compat.pluginApi` en `openclaw.build.openclawVersion`.

## Controleren vóór installatie

Gebruik vóór de installatie de ClawHub-webpagina of detailopdrachten van de CLI om
metadata, bronlinks, versies, wijzigingslogboeken en de scanstatus te controleren:

```bash
clawhub inspect @openclaw/demo
clawhub package inspect <package>
```

Openbare vermeldingen tonen de nieuwste scanstatus. Releases die door
moderatie worden vastgehouden of geblokkeerd, kunnen verborgen blijven in zoek- en installatievoorzieningen totdat het probleem is opgelost.
