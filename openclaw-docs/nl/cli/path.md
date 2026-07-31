---
read_when:
    - Je wilt vanuit de terminal een leaf in een werkruimtebestand lezen of schrijven
    - Je schrijft scripts die werken met de werkruimtestatus en wilt een stabiel adresseringsschema dat onafhankelijk is van het type
    - Je debugt een `oc://`-pad (valideer de syntaxis en kijk waarnaar het wordt omgezet)
summary: CLI-referentie voor `openclaw path` (werkruimtebestanden inspecteren en bewerken via het adresseringsschema `oc://`)
title: Pad
x-i18n:
    generated_at: "2026-07-27T05:46:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

Shelltoegang tot het `oc://`-adresseringsschema: één padsyntaxis met dispatch op type
voor het inspecteren en bewerken van adresseerbare werkruimtebestanden (markdown, jsonc,
jsonl, yaml/yml/lobster). Self-hosters, pluginauteurs en editorextensies
gebruiken deze om een specifieke locatie te lezen, zoeken of bij te werken zonder handmatig
een parser per bestand te schrijven.

`path` wordt geleverd door de gebundelde optionele `oc-path`-plugin. Schakel deze vóór
het eerste gebruik in:

```bash
openclaw plugins enable oc-path
```

De CLI-werkwoorden weerspiegelen het adresseringsmodel:

- `resolve` is concreet en levert één overeenkomst op.
- `find` is het werkwoord voor meerdere overeenkomsten bij jokertekens, unions, predicaten en
  positionele uitbreiding.
- `set` accepteert alleen concrete paden of invoegmarkeringen; patronen met jokertekens
  worden vóór het schrijven geweigerd.
- `validate` parseert een pad zonder bestandssysteemtoegang.
- `emit` voert een bestand heen en terug door parseren + uitvoeren (diagnose van bytegetrouwheid).

## Waarom dit gebruiken

De status van OpenClaw is verspreid over handmatig bewerkte markdown, JSONC-configuratie
met commentaar, alleen-toevoegen-JSONL-logboeken en YAML-workflow-/specificatiebestanden. Scripts, hooks
en agents hebben vaak één kleine waarde uit die bestanden nodig: een frontmatter-sleutel, een
plugininstelling, een veld van een logrecord, een YAML-stap of een opsommingsteken onder een
benoemde sectie.

`openclaw path` geeft deze aanroepers een stabiel adres in plaats van een eenmalige
grep, regex of parser per bestandstype. Hetzelfde `oc://`-pad kan vanuit de terminal worden gevalideerd,
opgelost, doorzocht, als proef worden uitgevoerd en geschreven, waardoor gerichte
automatisering controleerbaar en herhaalbaar blijft. De rest van het bestand blijft behouden, zodat
het schrijven van één blad de opmerkingen, regeleinden of nabijgelegen
opmaak niet verstoort.

Gebruik dit wanneer het gewenste onderdeel een logisch adres heeft, maar de bestandsvorm
varieert:

- Een hook leest één instelling uit JSONC met commentaar zonder opmerkingen te verliezen wanneer
  de waarde wordt teruggeschreven.
- Een onderhoudsscript vindt elk overeenkomend gebeurtenisveld in een JSONL-logboek
  zonder het volledige logboek in een aangepaste parser te laden.
- Een editor springt op basis van een slug naar een markdownsectie of opsommingsitem en geeft vervolgens
  exact de regel weer waarnaar het adres is opgelost.
- Een agent voert eerst een proefbewerking van een klein deel van de werkruimte uit voordat deze wordt toegepast, waarbij de
  gewijzigde bytes zichtbaar zijn tijdens de review.

Gebruik `openclaw path` niet voor gewone bewerkingen van volledige bestanden, uitgebreide configuratiemigraties of
geheugenspecifieke schrijfbewerkingen; gebruik daarvoor de opdracht of plugin van de eigenaar. `path`
is bedoeld voor kleine, adresseerbare bestandsbewerkingen waarbij een herhaalbare terminalopdracht
beter is dan nog een speciaal gebouwde parser.

## Gebruik

Lees één waarde uit een handmatig bewerkt configuratiebestand:

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

Bekijk een schrijfbewerking zonder de schijf te wijzigen:

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

Zoek overeenkomende records in een alleen-toevoegen-JSONL-logboek:

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Adresseer een instructie in markdown op sectie en item in plaats van op
regelnummer:

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

Valideer een pad in CI of een preflightscript voordat het script leest of
schrijft:

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

Deze opdrachten zijn bedoeld om naar shellscripts te kunnen worden gekopieerd. Gebruik `--json` wanneer
een aanroeper gestructureerde uitvoer nodig heeft en `--human` wanneer iemand het resultaat
inspecteert.

## Werking

1. Parseert het `oc://`-adres in posities: bestand, sectie, item, veld en een
   optionele sessiequery.
2. Kiest de adapter voor het bestandstype op basis van de extensie van het doel (`.md`, `.jsonc`,
   `.json`, `.jsonl`, `.ndjson`, `.yaml`, `.yml`, `.lobster`).
3. Lost de posities op aan de hand van de structuur van dat bestandstype: markdown-
   koppen/items, JSONC-objectsleutels/array-indexen, JSONL-regelrecords of
   YAML-map-/sequentieknooppunten.
4. Voor `set` voert de adapter de bewerkte bytes uit, zodat onaangeroerde delen
   van het bestand hun opmerkingen, regeleinden en nabijgelegen opmaak behouden waar
   het bestandstype dit ondersteunt.

`resolve` en `set` vereisen één concreet doel. `find` is het verkennende
werkwoord: het breidt jokertekens, unions, predicaten en rangnummers uit tot de concrete
overeenkomsten die je kunt inspecteren voordat je er één kiest om te schrijven.

## Subopdrachten

| Subopdracht              | Doel                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | De concrete overeenkomst op het pad weergeven (of "niet gevonden").                      |
| `find <pattern>`        | Overeenkomsten voor een pad met jokerteken / union / predicaat opsommen.                  |
| `set <oc-path> <value>` | Een blad of invoegdoel op een concreet pad schrijven. Ondersteunt `--dry-run`.  |
| `validate <oc-path>`    | Alleen parseren; de structurele uitsplitsing weergeven (bestand / sectie / item / veld). |
| `emit <file>`           | Een bestand heen en terug door parseren + uitvoeren voeren (diagnose van bytegetrouwheid).          |

## Globale vlaggen

| Vlag            | Van toepassing op                       | Doel                                                                  |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`, `find`, `set`, `emit` | De bestandspositie ten opzichte van deze map oplossen (standaard: `process.cwd()`). |
| `--file <path>` | `resolve`, `find`, `set`, `emit` | Het opgeloste pad van de bestandspositie overschrijven (absolute toegang).                |
| `--json`        | alle                              | JSON-uitvoer afdwingen (standaard wanneer stdout geen TTY is).                    |
| `--human`       | alle                              | Voor mensen leesbare uitvoer afdwingen (standaard wanneer stdout een TTY is).                       |
| `--value-json`  | `set`                            | `<value>` als JSON parseren voor vervanging van JSON/JSONC/JSONL-bladeren.           |
| `--dry-run`     | `set`                            | De bytes weergeven die zouden worden geschreven, zonder ze te schrijven.                   |
| `--diff`        | `set` (vereist `--dry-run`)     | Een unified diff weergeven in plaats van de volledige bytes.                          |

`validate` accepteert alleen `--json` / `--human`; deze opdracht gebruikt het bestandssysteem niet, dus
`--cwd` en `--file` zijn niet van toepassing.

## `oc://`-syntaxis

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

Regels voor posities: `field` vereist `item` en `item` vereist `section`. Voor
alle vier posities geldt:

- **Segmenten tussen aanhalingstekens** — `"a/b.c"` blijft behouden bij `/`- en `.`-scheidingstekens. De inhoud is
  byteletterlijk; `"` en `\` zijn niet toegestaan binnen aanhalingstekens. Ook de bestandspositie
  houdt rekening met aanhalingstekens: `oc://"skills/email-drafter"/Tools/$last` behandelt
  `skills/email-drafter` als één bestandspad.
- **Predicaten** — `[k=v]`, `[k!=v]`, `[k<v]`, `[k<=v]`, `[k>v]`, `[k>=v]`.
  Numerieke operatoren vereisen dat beide zijden naar eindige getallen kunnen worden geconverteerd.
- **Unions** — `{a,b,c}` komt overeen met elk van de alternatieven.
- **Jokertekens** — `*` (één subsegment) en `**` (nul of meer,
  recursief). `find` accepteert deze; `resolve` en `set` weigeren ze als
  ambigu.
- **Positioneel** — `$first` / `$last` worden opgelost naar de eerste / laatste index of
  gedeclareerde sleutel.
- **Rangnummer** — `#N` voor de N-de overeenkomst in documentvolgorde.
- **Invoegmarkeringen** — `+`, `+key`, `+nnn` voor invoeging op sleutel / index
  (te gebruiken met `set`).
- **Sessiebereik** — `?session=cron-daily` enzovoort. Staat los van de nesting van posities.
  Sessiewaarden zijn onbewerkt en niet procentgedecodeerd; ze mogen geen besturings-
  tekens of gereserveerde queryscheidingstekens bevatten (`?`, `&`, `%`).

Gereserveerde tekens (`?`, `&`, `%`) buiten segmenten tussen aanhalingstekens, predicaatsegmenten of unionsegmenten
worden geweigerd. Besturingstekens (U+0000-U+001F, U+007F) worden
overal geweigerd, inclusief in de `session`-querywaarde.

`formatOcPath(parseOcPath(path)) === path` is gegarandeerd voor canonieke paden.
Niet-canonieke queryparameters worden genegeerd, behalve de eerste niet-lege
`session=`-waarde.

Harde limieten: een pad is beperkt tot 4096 bytes, maximaal 4 posities (bestand/sectie/item/
veld), maximaal 64 met punten gescheiden subsegmenten per positie en maximaal 256 geneste
traversalniveaus voor diepe JSON-paden. Daarnaast wordt elk JSONC/JSON-invoerbestand
groter dan 16 MiB geweigerd met een parsediagnose in plaats van geparseerd, voor
elk werkwoord dat dat bestand laadt.

## Adressering per bestandstype

| Type          | Bestandsextensies             | Adresseringsmodel                                                                                    |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | H2-secties op slug, opsommingsitems op slug of `#N`, frontmatter via `[frontmatter]`.                 |
| JSONC/JSON    | `.jsonc`, `.json`           | Objectsleutels en array-indexen; punten splitsen geneste subsegmenten, tenzij ze tussen aanhalingstekens staan.                        |
| JSONL         | `.jsonl`, `.ndjson`         | Adressen van regels op het hoogste niveau (`L1`, `L2`, `$first`, `$last`), gevolgd door afdaling in JSONC-stijl binnen de regel. |
| YAML/.lobster | `.yaml`, `.yml`, `.lobster` | Mapsleutels en sequentie-indexen; opmerkingen en flowstijl worden verwerkt door de YAML-document-API.        |

`resolve` retourneert een gestructureerde overeenkomst: `root`, `node`, `leaf` of
`insertion-point`, met een regelnummer op basis van 1. Bladwaarden worden beschikbaar gesteld als
tekst plus een `leafType`, zodat pluginauteurs voorbeelden kunnen weergeven zonder
afhankelijk te zijn van de AST-vorm per type.

## Mutatiecontract

`set` schrijft één concreet doel:

- Markdown-frontmatterwaarden en `- key: value`-itemvelden zijn string-
  bladeren. Markdown-invoegingen voegen secties, frontmatter-sleutels of sectie-
  items toe en renderen een canonieke Markdown-vorm voor het gewijzigde bestand. Sectie-
  inhoud kan niet als geheel worden geschreven via `set`.
- Bij het schrijven van JSONC-bladeren wordt de stringwaarde geconverteerd naar het bestaande bladtype
  (`string`, eindige `number`, `true`/`false` of `null`). Gebruik `--value-json`
  wanneer een vervanging van een JSONC-/JSON-/JSONL-blad `<value>` als JSON moet parseren en
  de structuur mag wijzigen, bijvoorbeeld wanneer een verkorte stringnotatie voor een geheime referentie wordt vervangen door een
  object. Bij invoegingen in JSONC-objecten en -arrays wordt `<value>` als JSON geparseerd en wordt
  het `jsonc-parser`-bewerkingspad gebruikt voor gewone schrijfbewerkingen van bladeren, waarbij opmerkingen
  en nabijgelegen opmaak behouden blijven.
- Bij het schrijven van JSONL-bladeren wordt binnen een regel geconverteerd zoals bij JSONC. Bij vervanging
  van een volledige regel en bij toevoegen wordt `<value>` als JSON geparseerd. Gerenderde JSONL behoudt de
  dominante LF-/CRLF-conventie voor regeleinden van het bestand (meerderheidsbesluit over alle
  regeleinden in het bestand, zodat een bestand met voornamelijk CRLF CRLF blijft gebruiken, zelfs met enkele afwijkende LF's).
- Bij het schrijven van YAML-bladeren wordt geconverteerd naar het bestaande scalaire type (`string`, eindige
  `number`, `true`/`false` of `null`). YAML-invoegingen gebruiken de document-API van het meegeleverde
  `yaml`-pakket voor updates van mappings/reeksen. Ongeldige YAML-
  documenten met parserfouten worden vóór wijziging geweigerd met
  `parse-error`.

Gebruik `--dry-run` vóór voor gebruikers zichtbare schrijfbewerkingen wanneer de exacte bytes van belang zijn. JSONC-
en YAML-bewerkingen passen het bestaande document aan (via `jsonc-parser` of de
document-API van `yaml`), zodat onaangeraakte bytes doorgaans behouden blijven; bij elke bewerking bouwt Markdown het bestand
opnieuw op vanuit de geparseerde structuur, waardoor bijkomstige
opmaak buiten het gewijzigde blad kan worden genormaliseerd. Voeg `--diff` toe wanneer je het voorbeeld
als een gerichte voor/na-patch wilt zien in plaats van het volledige gerenderde bestand.

## Voorbeelden

```bash
# Een pad valideren (geen toegang tot het bestandssysteem)
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# Een blad lezen
openclaw path resolve 'oc://gateway.jsonc/version'

# Zoeken met jokertekens
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# Een schrijfbewerking simuleren
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# Een schrijfbewerking simuleren als een unified diff
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# De schrijfbewerking toepassen
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# Bytegetrouwe roundtrip (diagnostiek)
openclaw path emit ./AGENTS.md
```

Meer grammaticavoorbeelden:

```bash
# Sleutels tussen aanhalingstekens plaatsen die / of . bevatten
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# Diepe JSON-/JSONC-paden kunnen slashsegmenten gebruiken; deze worden genormaliseerd naar subsegmenten met punten
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# Een JSONC-blad vervangen door een geparseerd object
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# Zoeken met een predicaat in JSONC-kinderen
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# Invoegen in een JSONC-array
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# Een sleutel in een JSONC-object invoegen
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# Een JSONL-gebeurtenis toevoegen
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# De laatste JSONL-waarderegel oplossen
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# Een YAML-workflowstap oplossen
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# Een YAML-scalaire waarde bijwerken
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# Markdown-frontmatter adresseren
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# Markdown-frontmatter invoegen
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# Markdown-itemvelden zoeken
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# Een sessiegebonden pad valideren
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## Recepten per bestandstype

Dezelfde vijf werkwoorden werken voor alle typen; het adresseringsschema kiest op basis van
de bestandsextensie.

### Markdown

```text
<!-- frontmatter.md -->
---
name: opsteller
description: agent voor het opstellen van e-mails
tier: kern
---
## Hulpmiddelen
- gh: GitHub-CLI
- curl: HTTP-client
- send_email: ingeschakeld
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
blad @ L4: "kern" (string)

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
blad @ L9: "GitHub-CLI" (string)

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
3 overeenkomsten voor oc://x.md/tools/*:
  oc://x.md/tools/gh           →  knooppunt @ L9 [md-item]
  oc://x.md/tools/curl         →  knooppunt @ L10 [md-item]
  oc://x.md/tools/send-email   →  knooppunt @ L11 [md-item]
```

Het predicaat `[frontmatter]` adresseert het YAML-frontmatterblok; `tools`
komt via de slug overeen met de kop `## Tools`, en itembladeren behouden hun slugvorm,
zelfs wanneer de bron underscores gebruikt (`send_email` wordt `send-email`).

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
blad @ L4: "true" (boolean)

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: zou 142 bytes schrijven naar /…/config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC-bewerkingen verlopen via `jsonc-parser`, zodat opmerkingen en witruimte een
`set` overleven. Voer eerst uit met `--dry-run` om de bytes te controleren voordat je de wijziging definitief maakt.
`.json`-bestanden gebruiken dezelfde adapter en hetzelfde bewerkingspad als `.jsonc`.

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
1 overeenkomst voor oc://session.jsonl/[event=action]/userId:
  oc://session.jsonl/L2/userId  →  blad @ L2: "u1" (string)

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
blad @ L2: "2" (number)
```

Elke regel is een record. Adresseer met een predicaat (`[event=action]`) wanneer je
het regelnummer niet weet, of met het canonieke `LN`-segment wanneer je het wel weet.
`.ndjson`-bestanden gebruiken dezelfde adapter als `.jsonl`.

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
blad @ L3: "fetch" (string)

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: zou 99 bytes schrijven naar /…/workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML gebruikt de `Document`-API van het `yaml`-pakket in plaats van een zelfgeschreven
parser, zodat gewone parseer-/render-roundtrips opmerkingen en de door de auteur gekozen
vorm behouden, terwijl opgeloste paden hetzelfde model voor mapping-sleutels/reeksindexen gebruiken als
JSONC. Dezelfde adapter verwerkt `.yaml`-, `.yml`- en `.lobster`-bestanden.

## Naslag voor subopdrachten

### `resolve <oc-path>`

Lees één blad of knooppunt. Jokertekens worden geweigerd — gebruik daarvoor `find`.
Eindigt met `0` bij een overeenkomst, `1` bij een reguliere misser en `2` bij een parseerfout of geweigerd
patroon.

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

Som elke overeenkomst voor een jokerteken-/predicaat-/uniepatroon op. Eindigt met `0`
bij ten minste één overeenkomst en `1` bij nul overeenkomsten. Jokertekens voor bestandsposities worden geweigerd met
`OC_PATH_FILE_WILDCARD_UNSUPPORTED` — geef een concreet bestand door (globben over meerdere
bestanden is een toekomstige functie).

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

Schrijf een blad. Combineer met `--dry-run` om een voorbeeld te bekijken van de bytes die zouden worden
geschreven zonder het bestand te wijzigen. Voeg `--diff` toe voor een voorbeeld als unified diff.
Eindigt met `0` na een geslaagde schrijfbewerking, `1` als de onderlaag weigert (bijvoorbeeld wanneer
een sentinel-beveiliging wordt geactiveerd) en `2` bij parseerfouten.

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

De invoegmarkering `+key` maakt het genoemde kind aan als dit nog niet
bestaat; `+nnn` en de losse `+` werken respectievelijk voor geïndexeerd invoegen en toevoegen.

### `validate <oc-path>`

Controleert alleen het parseren. Geen toegang tot het bestandssysteem. Handig wanneer je wilt bevestigen dat een
sjabloonpad correct is gevormd voordat je variabelen vervangt, of wanneer je
de structurele uitsplitsing nodig hebt voor foutopsporing:

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
geldig: oc://AGENTS.md/tools/gh
  bestand: AGENTS.md
  sectie:  tools
  item:    gh
```

Eindigt met `0` indien geldig, `1` indien ongeldig (met een gestructureerde `code` en
`message`) en `2` bij argumentfouten.

### `emit <file>`

Voer een bestand door de parser en emitter voor het betreffende type voor een roundtrip. De uitvoer hoort
byte-identiek te zijn aan de invoer bij een geldig bestand; een afwijking wijst op een
parserfout of een geactiveerde sentinel. Handig voor het opsporen van problemen met het gedrag van de onderlaag bij
praktijkinvoer.

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## Afsluitcodes

| Code | Betekenis                                                                  |
| ---- | -------------------------------------------------------------------------- |
| `0`  | Geslaagd. (`resolve` / `find`: ten minste één overeenkomst. `set`: schrijven geslaagd.) |
| `1`  | Geen overeenkomst, of `set` geweigerd door de onderlaag (geen fout op systeemniveau). |
| `2`  | Argument- of parseerfout.                                                   |

## Uitvoermodus

`openclaw path` houdt rekening met TTY: voor mensen leesbare uitvoer in een terminal, JSON wanneer
stdout via een pipe wordt doorgegeven of wordt omgeleid. `--json` en `--human` overschrijven de
automatische detectie.

## Opmerkingen

- `set` schrijft bytes via het emit-pad van de onderliggende laag, dat automatisch de
  bewaking met de redactiesentinel toepast. Een leaf die
  `__OPENCLAW_REDACTED__` bevat (letterlijk of als subtekenreeks), wordt tijdens het
  schrijven geweigerd.
- Voor JSONC-parsing en het bewerken van leafs wordt de Plugin-lokale afhankelijkheid `jsonc-parser`
  gebruikt, zodat opmerkingen en opmaak bij normale schrijfbewerkingen van leafs behouden blijven
  in plaats van via een handmatig gebouwde parser en een pad voor opnieuw renderen te lopen.
- `path` houdt geen rekening met het bijhouden of herstellen van de laatst bekende werkende configuratie (LKG);
  die levenscyclus wordt elders beheerd. Als een bestand dat je via `path` bewerkt
  ook door LKG wordt bijgehouden, bepaalt de volgende configuratielezing of het wordt overgenomen of
  hersteld; behandel een bewerking via `path` hetzelfde als elke andere directe schrijfbewerking naar
  dat bestand.

## Gerelateerd

- [CLI-referentie](/nl/cli)
