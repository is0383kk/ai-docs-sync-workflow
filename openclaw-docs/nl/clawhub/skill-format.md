---
read_when:
    - Skills publiceren
    - Publicatiefouten opsporen
summary: Indeling van de Skills-map, vereiste bestanden, ondersteunende artefacten, limieten.
x-i18n:
    generated_at: "2026-07-27T05:04:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# Skill-indeling

## Op schijf

Een Skill is een map.

Vereist:

- `SKILL.md` (of `skill.md`; verouderde `skills.md` wordt ook geaccepteerd)

Optioneel:

- alle ondersteunende reguliere bestanden (zie ‘Skill-bestanden’)
- `.clawhubignore` (negeerpatronen voor publicatie, verouderde `.clawdhubignore`)
- `.gitignore` (wordt ook gehonoreerd)

## Importeren vanuit GitHub

De GitHub-importfunctie op het web is strenger dan lokaal publiceren/synchroniseren. Deze ontdekt alleen
`SKILL.md`- of verouderde `skills.md`-bestanden in openbare repositories die geen fork zijn en eigendom zijn van
het aangemelde GitHub-account. Privérepositories, forks,
gearchiveerde/uitgeschakelde repositories en openbare repositories van derden worden niet geïmporteerd.

Metadata van lokale installatie (geschreven door de CLI):

- `<skill>/.clawhub/origin.json` (verouderde `.clawdhub`)

Installatiestatus van de werkmap (geschreven door de CLI):

- `<workdir>/.clawhub/lock.json` (verouderde `.clawdhub`)

## `SKILL.md`

- Markdown met optionele YAML-frontmatter.
- De server haalt tijdens het publiceren metadata uit de frontmatter.
- `description` wordt gebruikt als samenvatting van de Skill in de gebruikersinterface/zoekfunctie.

Voor overdraagbare Agent Skills moet `name` overeenkomen met de bovenliggende map en
1–64 kleine letters, cijfers of koppeltekens bevatten. ClawHub houdt de routeerbare slug en
de weergavenaam in de catalogus gescheiden, zodat bestaande namen uit andere clients
publiceerbaar blijven en niet stilzwijgend worden herschreven. In cataloguslijsten kunnen lange namen
visueel worden ingekort zonder de opgeslagen naam te wijzigen.

## Frontmatter-metadata

Skill-metadata wordt gedeclareerd in de YAML-frontmatter bovenaan je `SKILL.md`. Hiermee wordt aan het register (en de beveiligingsanalyse) doorgegeven wat je Skill nodig heeft om te worden uitgevoerd.

### Basisfrontmatter

```yaml
---
name: my-skill
description: Korte samenvatting van wat deze Skill doet.
version: 1.0.0
---
```

### Runtime-metadata (`metadata.openclaw`)

Declareer de runtimevereisten van je Skill onder `metadata.openclaw` (aliassen: `metadata.clawdbot`, `metadata.clawdis`).

```yaml
---
name: my-skill
description: Beheer taken via de Todoist-API.
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
---
```

Gebruik `requires.env` voor omgevingsvariabelen die aanwezig moeten zijn voordat de Skill kan worden uitgevoerd. Gebruik `envVars` wanneer je metadata per variabele nodig hebt, waaronder optionele variabelen met `required: false`.

### Volledig veldenoverzicht

| Veld               | Type       | Beschrijving                                                                                                                                 |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | Vereiste omgevingsvariabelen die je Skill verwacht.                                                                                          |
| `requires.bins`    | `string[]` | CLI-binaire bestanden die allemaal geïnstalleerd moeten zijn.                                                                                |
| `requires.anyBins` | `string[]` | CLI-binaire bestanden waarvan er ten minste één aanwezig moet zijn.                                                                          |
| `requires.config`  | `string[]` | Paden naar configuratiebestanden die je Skill leest.                                                                                         |
| `primaryEnv`       | `string`   | De belangrijkste omgevingsvariabele voor aanmeldgegevens van je Skill.                                                                       |
| `envVars`          | `array`    | Declaraties van omgevingsvariabelen met `name`, optionele `required` en optionele `description`. Stel `required: false` in voor optionele omgevingsvariabelen. |
| `always`           | `boolean`  | Als `true`, is de Skill altijd actief (geen expliciete installatie nodig).                                                                  |
| `skillKey`         | `string`   | Overschrijf de aanroepsleutel van de Skill.                                                                                                  |
| `emoji`            | `string`   | Weergave-emoji voor de Skill.                                                                                                                |
| `homepage`         | `string`   | URL naar de homepage of documentatie van de Skill.                                                                                           |
| `os`               | `string[]` | Besturingssysteembeperkingen (bijv. `["macos"]`, `["linux"]`).                                                                                |
| `install`          | `array`    | Installatiespecificaties voor afhankelijkheden (zie hieronder).                                                                              |
| `nix`              | `object`   | Specificatie voor de Nix-Plugin (zie README).                                                                                                |
| `config`           | `object`   | Clawdbot-configuratiespecificatie (zie README).                                                                                               |

### Installatiespecificaties

Als je Skill geïnstalleerde afhankelijkheden nodig heeft, declareer je deze in de array `install`:

```yaml
metadata:
  openclaw:
    install:
      - kind: brew
        formula: jq
        bins: [jq]
      - kind: node
        package: typescript
        bins: [tsc]
```

Ondersteunde installatietypen: `brew`, `node`, `go`, `uv`.

### Optionele omgevingsvariabelen

Declareer optionele omgevingsvariabelen onder `metadata.openclaw.envVars` en stel `required: false` in. Voeg geen optionele vermeldingen toe aan `requires.env`, omdat `requires.env` betekent dat de Skill zonder deze variabelen niet kan worden uitgevoerd.

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Todoist-API-token dat wordt gebruikt voor geauthenticeerde verzoeken.
      - name: TODOIST_PROJECT_ID
        required: false
        description: Optionele standaardproject-ID wanneer de gebruiker er geen opgeeft.
```

### Waarom dit belangrijk is

De beveiligingsanalyse van ClawHub controleert of wat je Skill declareert overeenkomt met wat deze daadwerkelijk doet. Als je code verwijst naar `TODOIST_API_KEY`, maar dit in je frontmatter niet declareert onder `requires.env`, `primaryEnv` of `envVars`, markeert de analyse dit als niet-overeenkomende metadata. Nauwkeurige declaraties helpen je Skill de beoordeling te doorstaan en helpen gebruikers begrijpen wat ze installeren.

### Voorbeeld: volledige frontmatter

```yaml
---
name: todoist-cli
description: Beheer Todoist-taken, -projecten en -labels vanaf de opdrachtregel.
version: 1.2.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Todoist-API-token.
      - name: TODOIST_PROJECT_ID
        required: false
        description: Optionele standaardproject-ID.
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## Skill-bestanden

Publiceren accepteert alle reguliere bestanden in de Skill-map, ongeacht de extensie. Negeerbestanden,
verborgen paden, symbolische koppelingen, macOS-metadata en serverzijdige groottelimieten blijven van toepassing.

- Bestanden met een begrensde grootte die geldige UTF-8 bevatten, kunnen als ontsnapte platte tekst worden bekeken en worden opgenomen
  in begrensde tekstanalyse.
- Andere bestanden behouden exact hun bytes en kunnen worden gedownload.
- Beveiligingsscanners ontvangen het volledige opgeslagen artefact; tekstdetectie betreft weergave en
  analyse en is geen uploadtoelatingslijst.

Limieten (serverzijde):

- Totale bundelgrootte: 50MB.
- Insluitingstekst omvat `SKILL.md` + maximaal circa 40 begrensde UTF-8-bestanden (limiet op basis van beste inspanning).

## Slugs

- Standaard afgeleid van de mapnaam.
- Pakketbereiken moeten exact overeenkomen met de ClawHub-handle van de uitgever. Uitgevershandles mogen kleine letters, cijfers, koppeltekens, punten en underscores bevatten; ze moeten beginnen en eindigen met een kleine letter of cijfer.
- Pakketslugs moeten uit kleine letters bestaan en npm-veilig zijn, bijvoorbeeld `@example.tools/demo-plugin` of `demo-plugin`.

## Versiebeheer + tags

- Elke publicatie maakt een nieuwe versie (semver).
- Tags zijn tekenreeksverwijzingen naar een versie; `latest` wordt vaak gebruikt.

## Licentie

- Alle Skills die op ClawHub worden gepubliceerd, vallen onder de licentie `MIT-0`.
- Iedereen mag gepubliceerde Skills gebruiken, wijzigen en herdistribueren, ook commercieel.
- Naamsvermelding is niet vereist.
- Voeg geen conflicterende licentievoorwaarden toe in `SKILL.md`; ClawHub ondersteunt geen licentieoverschrijvingen per Skill.

## Betaalde Skills

- ClawHub ondersteunt geen betaalde Skills, prijzen per Skill, betaalmuren of inkomstendeling.
- Voeg geen prijsmetadata toe aan `SKILL.md`; dit maakt geen deel uit van de Skill-indeling en maakt een gepubliceerde Skill niet betaald.
- Als je Skill integreert met een betaalde service van derden, documenteer dan duidelijk de externe kosten en het vereiste account in de Skill-instructies en omgevingsdeclaraties (`requires.env` voor vereiste variabelen, of `envVars` met `required: false` voor optionele variabelen).
