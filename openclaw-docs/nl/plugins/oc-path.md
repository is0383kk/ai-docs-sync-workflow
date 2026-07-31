---
read_when:
    - Je wilt vanuit de terminal één enkel eindelement in een werkruimtebestand bekijken of bewerken
    - Je schrijft scripts voor de werkruimtestatus en hebt een stabiel, typeonafhankelijk adresseringsschema nodig
    - Je beslist of je de optionele `oc-path`-Plugin wilt inschakelen op een zelfgehoste Gateway
summary: 'Gebundelde `oc-path`-Plugin: levert de `openclaw path`-CLI voor het adresseringsschema voor `oc://`-werkruimtebestanden'
title: OC Path-Plugin
x-i18n:
    generated_at: "2026-07-27T06:01:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb7bb1aacd37e5cc9c391372b871dc519f4048232d93a0016138ae00a6985a59
    source_path: plugins/oc-path.md
    workflow: 16
---

De gebundelde `oc-path`-plugin voegt de [`openclaw path`](/nl/cli/path)-CLI toe voor het
`oc://`-adresseringsschema voor werkruimtebestanden. Deze wordt meegeleverd in de OpenClaw-repo onder
`extensions/oc-path/`, maar is opt-in: na installatie/build blijft de plugin inactief totdat je
deze inschakelt.

`oc://`-adressen verwijzen naar één enkel blad (of een set bladeren met jokertekens) binnen
een werkruimtebestand. De plugin ondersteunt vier bestandstypen:

- **markdown** (`.md`): frontmatter, secties, items, velden
- **jsonc** (`.jsonc`, `.json`): opmerkingen en opmaak blijven behouden
- **jsonl** (`.jsonl`, `.ndjson`): regelgeoriënteerde records
- **yaml** (`.yaml`, `.yml`, `.lobster`): map-/reeks-/scalaire knooppunten via de
  `Document`-API van het pakket `yaml`

Self-hosters en editoruitbreidingen gebruiken de CLI om één blad te lezen of te schrijven
zonder rechtstreeks scripts voor de SDK te maken; agents en hooks behandelen deze als een
deterministische onderlaag, zodat bytegetrouwe round-trips en de bewaking met
redactiesentinel uniform voor alle typen gelden. Zie de
[CLI-referentie](/nl/cli/path) voor de volledige grammatica, de lijst met vlaggen per opdracht en
uitgewerkte voorbeelden per bestandstype; deze pagina beschrijft waarom en hoe je de
plugin inschakelt.

## Waarom inschakelen

Schakel `oc-path` in wanneer scripts, hooks of lokale agenttools moeten verwijzen naar
een specifiek onderdeel van de werkruimtestatus zonder een speciale parser per bestandsvorm. Eén
`oc://`-adres kan een markdown-frontmatter-sleutel, een sectie-item, een
JSONC-configuratieblad, een JSONL-gebeurtenisveld of een YAML-workflowstap aanduiden.

Dat is belangrijk voor onderhoudsworkflows waarin de wijziging klein,
controleerbaar en herhaalbaar moet blijven: inspecteer één waarde, zoek overeenkomende records, voer een
proefschrijfopdracht uit en pas vervolgens alleen dat blad toe, terwijl opmerkingen, regeleinden en
omliggende opmaak ongewijzigd blijven.

Veelvoorkomende redenen om de plugin in te schakelen:

- **Lokale automatisering**: shellscripts zoeken of werken één werkruimtewaarde bij
  met `openclaw path … --json`, in plaats van afzonderlijke parseercode voor markdown, JSONC,
  JSONL en YAML te bevatten.
- **Voor agents zichtbare bewerkingen**: een agent toont vóór het schrijven een proefdiff voor één geadresseerd
  blad, die eenvoudiger te beoordelen is dan een vrije
  herschrijving van het bestand.
- **Editorintegraties**: een editor koppelt `oc://AGENTS.md/tools/gh` aan het
  exacte markdown-knooppunt en regelnummer zonder te gokken op basis van koptekst.
- **Diagnostiek**: `emit` voert een round-trip van een bestand uit via de parser en emitter,
  zodat je kunt controleren of een bestandstype byte-stabiel is voordat je vertrouwt op
  geautomatiseerde bewerkingen.

```bash
# Is de GitHub-plugin ingeschakeld in deze configuratie?
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --json

# Welke namen van toolaanroepen komen voor in dit sessielogboek?
openclaw path find 'oc://session.jsonl/[event=tool_call]/name' --json

# Welke bytes zou deze kleine configuratiebewerking schrijven?
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

`oc-path` is bewust niet verantwoordelijk voor semantiek op een hoger niveau. Geheugenplugins
blijven verantwoordelijk voor schrijfbewerkingen naar het geheugen, configuratieopdrachten blijven verantwoordelijk voor het volledige configuratiebeheer
en herstel van de laatst bekende goede configuratie (LKG) blijft verantwoordelijk voor
herstel/promotie. `oc-path` is de beperkte adresserings- en bytebehoudende
bestandsbewerkingslaag waarop die tools op hoger niveau kunnen voortbouwen.

## Waar deze wordt uitgevoerd

De plugin wordt **in-process binnen de `openclaw`-CLI** uitgevoerd op de host waarop je
de opdracht aanroept. Hiervoor is geen actieve Gateway nodig en er worden geen
netwerksockets geopend; elke opdracht is een zuivere transformatie van een bestand dat je opgeeft.

De metadata van de plugin bevindt zich in `extensions/oc-path/openclaw.plugin.json`:

```json
{
  "id": "oc-path",
  "name": "OC Path",
  "activation": {
    "onStartup": false,
    "onCommands": ["path"]
  },
  "commandAliases": [{ "name": "path", "kind": "cli" }]
}
```

`onStartup: false` houdt de plugin buiten het opstartpad van de Gateway.
`commandAliases` en `activation.onCommands` geven de CLI opdracht de plugin
pas te laden wanneer je `openclaw path …` voor het eerst uitvoert, zodat installaties die
de opdracht nooit gebruiken geen kosten maken.

## Inschakelen

```bash
openclaw plugins enable oc-path
```

Start de Gateway opnieuw (als je er een uitvoert), zodat de manifestmomentopname de nieuwe
status overneemt. Afzonderlijke aanroepen van `openclaw path` werken direct op dezelfde host;
de CLI laadt de plugin op aanvraag.

Uitschakelen doe je met:

```bash
openclaw plugins disable oc-path
```

## Afhankelijkheden

Alle parserafhankelijkheden zijn lokaal voor de plugin; het inschakelen van `oc-path` haalt geen
nieuwe pakketten naar de kernruntime:

| Afhankelijkheid     | Doel                                                                |
| -------------- | ---------------------------------------------------------------------- |
| `commander`    | Koppeling van subopdrachten voor `resolve`, `find`, `set`, `validate`, `emit`.    |
| `jsonc-parser` | JSONC-parsing en bladbewerking waarbij opmerkingen en afsluitende komma's behouden blijven.     |
| `markdown-it`  | Markdown-tokenisatie voor het sectie-/item-/veldmodel.            |
| `yaml`         | YAML `Document`-parsing/-emissie/-bewerking waarbij opmerkingen en flowstijl behouden blijven. |

JSONL blijft handgeschreven: regelgeoriënteerde parsing is eenvoudiger dan welke
afhankelijkheid ook, en parsing per regel verloopt al via `jsonc-parser`.

## Wat deze biedt

| Oppervlak                        | Geleverd door                                             |
| ------------------------------ | ------------------------------------------------------- |
| `openclaw path`-CLI            | `extensions/oc-path/cli-registration.ts`                |
| `oc://`-parser/-formatter     | `extensions/oc-path/src/oc-path/oc-path.ts`             |
| Parsing/emissie/bewerking per type   | `extensions/oc-path/src/oc-path/{md,jsonc,jsonl,yaml}`  |
| Universeel zoeken/oplossen/instellen | `extensions/oc-path/src/oc-path/{resolve,find,edit}.ts` |
| Bewaking met redactiesentinel       | `extensions/oc-path/src/oc-path/sentinel.ts`            |

De CLI is momenteel het enige openbare oppervlak. De opdrachten van de onderlaag zijn privé voor
de plugin; gebruikers gebruiken de CLI (of bouwen hun eigen plugin op basis van de
SDK).

## Relatie tot andere plugins

- **`memory-*`**: schrijfbewerkingen naar het geheugen verlopen via de geheugenplugins, niet via
  `oc-path`. `oc-path` is een algemene bestandsonderlaag; geheugenplugins voegen
  daar hun eigen semantiek aan toe.
- **LKG**: `path` kent het herstel van de laatst bekende goede configuratie niet. Als een
  bestand dat je via `path` bewerkt ook door LKG wordt gevolgd, bepaalt de volgende observatiecyclus van de configuratie
  of het wordt gepromoveerd of hersteld; behandel een `path`-bewerking
  hetzelfde als elke andere rechtstreekse schrijfbewerking naar dat bestand.

## Veiligheid

`set` schrijft onbewerkte bytes via het emissiepad van de onderlaag, dat automatisch de
bewaking met redactiesentinel toepast. Een blad met
`__OPENCLAW_REDACTED__` (letterlijk of als deeltekenreeks) wordt tijdens het schrijven geweigerd
met `OC_EMIT_SENTINEL`. De CLI verwijdert de letterlijke sentinel ook uit alle
menselijke of JSON-uitvoer en vervangt deze door `[REDACTED]`, zodat terminalopnamen
en pijplijnen de markering nooit lekken.

## Gerelateerd

- [CLI-referentie voor `openclaw path`](/nl/cli/path)
- [Plugins beheren](/nl/plugins/manage-plugins)
- [Plugins bouwen](/nl/plugins/building-plugins)
