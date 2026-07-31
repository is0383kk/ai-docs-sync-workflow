---
read_when:
    - Je wilt permanente kennis die verder gaat dan gewone MEMORY.md-notities
    - Je configureert de meegeleverde memory-wiki-Plugin
    - Je hebt afzonderlijke wiki-kluizen nodig voor agents in één Gateway
    - Je wilt wiki_search, wiki_get of de bridge-modus begrijpen
summary: 'memory-wiki: gecompileerde kennisopslag met herkomst, beweringen, dashboards en bridge-modus'
title: Geheugenwiki
x-i18n:
    generated_at: "2026-07-27T05:40:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki` is een gebundelde plugin die duurzame kennis omzet in een
navigeerbare wiki: deterministische pagina's, gestructureerde claims met bewijs,
herkomst, dashboards en machineleesbare samenvattingen.

Deze vervangt de Active Memory-plugin niet. Ophalen, promoveren, indexeren en
Dreaming blijven onder beheer van de geconfigureerde geheugenbackend
(`memory-core`, QMD, Honcho enzovoort). `memory-wiki` staat ernaast en zet
kennis om in een onderhouden wikilaag.

Schakel de plugin in voordat je de CLI, tools of runtime-integratie ervan gebruikt:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| Laag                 | Beheert                                                                           |
| -------------------- | --------------------------------------------------------------------------------- |
| Active Memory-plugin | Ophalen, semantisch zoeken, promoveren, Dreaming, geheugenruntime                  |
| `memory-wiki`        | Samengestelde wikipagina's, syntheses met rijke herkomst, dashboards, wiki zoeken/ophalen/toepassen |

Praktische regel:

- `memory_search` voor één brede ophaalronde over alle geconfigureerde corpora
- `wiki_search` / `wiki_get` wanneer je wikispecifieke rangschikking, herkomst of een geloofsstructuur op paginaniveau wilt
- `memory_search corpus=all` om beide lagen in één aanroep te doorzoeken, wanneer de Active Memory-plugin corpusselectie ondersteunt

Een gebruikelijke local-first-configuratie: QMD als de actieve geheugenbackend
voor het ophalen en `memory-wiki` in de modus `bridge` voor duurzame
gesynthetiseerde pagina's. Zie het voorbeeld voor QMD + bridge-modus onder
[Configuratie](#configuration).

Als de bridge-modus nul geëxporteerde artefacten meldt, stelt de Active Memory-plugin
momenteel geen openbare bridge-invoer beschikbaar. Voer eerst `openclaw wiki doctor` uit
en controleer vervolgens of de Active Memory-plugin openbare artefacten ondersteunt.

## Kluismodi

- `isolated` (standaard): eigen kluis, eigen bronnen, geen afhankelijkheid van de Active Memory-plugin. Gebruik dit voor een zelfstandige, gecureerde kennisopslag.
- `bridge`: leest openbare geheugenartefacten en gebeurtenislogboeken uit de Active Memory-plugin via openbare plugin-SDK-koppelingen. Gebruik dit om de geëxporteerde artefacten van de geheugenplugin samen te stellen zonder toegang tot interne privéonderdelen van de plugin.
- `unsafe-local`: expliciete nooduitgang voor lokale privépaden op dezelfde machine. Bewust experimenteel en niet-portabel; gebruik dit alleen als je de vertrouwensgrens begrijpt en specifiek lokale bestandssysteemtoegang nodig hebt die de bridge-modus niet kan bieden.

Kluismodus en kluisbereik zijn afzonderlijke keuzes:

- `vaultMode` bepaalt waar wiki-invoer vandaan komt.
- `vault.scope` bepaalt of alle agents één kluis gebruiken of elke agent een onderliggende kluis krijgt.

`vault.scope: "global"` is de standaard en behoudt het bestaande gedrag met één kluis.
Gebruik `vault.scope: "agent"` met de modus `isolated` of `bridge` wanneer
agents geen wikipagina's, samengestelde samenvattingen, zoekresultaten of
schrijfbewerkingen mogen delen. Agentbereik kan niet worden gecombineerd met
de modus `unsafe-local`, omdat die geconfigureerde privépaden geen invoer
zijn die eigendom is van agents. Configuratievalidatie wijst deze combinatie af.

De bridge-modus kan, afhankelijk van de configuratieschakelaar `bridge.*`,
het volgende indexeren:

- geëxporteerde geheugenartefacten (`indexMemoryRoot`)
- dagelijkse notities (`indexDailyNotes`)
- Dreaming-rapporten (`indexDreamReports`)
- geheugengebeurtenislogboeken (`followMemoryEvents`)

Wanneer de bridge-modus actief is en `bridge.readMemoryArtifacts` is ingeschakeld, worden
`openclaw wiki status`, `openclaw wiki doctor` en `openclaw wiki bridge
import` via de actieve Gateway
gerouteerd, zodat ze dezelfde context van de Active Memory-plugin zien als het
agent-/runtimegeheugen. Als bridge is uitgeschakeld of het lezen van artefacten
uitstaat, behouden deze opdrachten hun lokale/offlinegedrag.

## Kluisindeling

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

Beheerde inhoud blijft binnen gegenereerde blokken; menselijke notitieblokken
blijven bij regeneratie behouden.

- `sources/`: geïmporteerd bronmateriaal en pagina's op basis van bridge/unsafe-local
- `entities/`: duurzame zaken, personen, systemen, projecten, objecten
- `concepts/`: ideeën, abstracties, patronen, beleidsregels (ook de bestemming voor OKF-imports)
- `syntheses/`: samengestelde samenvattingen en onderhouden totalen
- `reports/`: gegenereerde dashboards

## Imports van Open Knowledge Format

```bash
openclaw wiki okf import ./bundles/ga4
```

Importeer een uitgepakte Open Knowledge Format-bundel in conceptpagina's van de
wiki. Dit is geschikt wanneer een gegevenscatalogus, documentatiecrawler of
verrijkingsagent al OKF produceert: behoud OKF als het draagbare
uitwisselingsartefact en laat `memory-wiki` dit omzetten in OpenClaw-eigen
conceptpagina's en samengestelde samenvattingen.

- niet-gereserveerde `.md`-bestanden zijn conceptdocumenten
- elk geïmporteerd concept vereist een niet-leeg frontmatter-veld `type`; ontbrekende `type` levert een waarschuwing `missing-type` op en het bestand wordt overgeslagen
- onbekende waarden voor `type` worden geaccepteerd als generieke concepten
- `index.md` en `log.md` zijn gereserveerd en worden nooit als concepten geïmporteerd
- defecte of externe Markdown-links blijven ongewijzigd

Geïmporteerde pagina's worden plat onder `concepts/` geplaatst, zodat
bestaande processen voor samenstellen, zoeken, ophalen en dashboards ze zonder
een tweede wikiboom kunnen gebruiken. Elke pagina behoudt de oorspronkelijke
OKF-concept-ID, het bronpad, `type`, `resource`,
`tags`, het tijdstempel en de volledige frontmatter van de producent.
Interne OKF-links worden herschreven naar de gegenereerde conceptpagina's van
de wiki en leveren ook gestructureerde `relationships`-vermeldingen met
`kind: okf-link` op.

## Gestructureerde claims en bewijs

Pagina's bevatten gestructureerde `claims`-frontmatter, niet alleen
vrije tekst. Elke claim kan `id`, `text`,
`status`, `confidence`, `evidence[]` en
`updatedAt` bevatten. Elke bewijsvermelding kan `kind`,
`sourceId`, `path`, `lines`,
`weight`, `confidence`, `privacyTier`,
`note` en `updatedAt` bevatten.

Hierdoor gedraagt de wiki zich als een geloofslaag en niet als een passieve
verzameling notities. Claims kunnen worden gevolgd, beoordeeld, betwist en
naar bronnen worden herleid.

## Entiteitsmetadata voor agents

Entiteitspagina's bevatten generieke routeringsmetadata die bruikbaar zijn
voor personen, teams, systemen, projecten of elk ander entiteitstype:

- `entityType`: bijvoorbeeld `person`, `team`, `system`, `project`
- `canonicalId`: stabiele identiteitssleutel voor aliassen en imports
- `aliases`: namen, handles of labels die naar dezelfde pagina verwijzen
- `privacyTier`: vrije tekenreeks; `public` wordt behandeld als geen beoordeling vereist, elke andere waarde (bijvoorbeeld `local-private`, `sensitive`, `confirm-before-use`) wordt gemarkeerd in `reports/privacy-review.md`
- `bestUsedFor` / `notEnoughFor`: compacte routeringsaanwijzingen
- `lastRefreshedAt`: tijdstempel voor bronvernieuwing, los van de bewerkingstijd van de pagina
- `personCard`: optionele persoonsgebonden routeringskaart (handles, sociale profielen, e-mailadressen, tijdzone, werkgebied, waarvoor te benaderen, waarvoor niet te benaderen, betrouwbaarheid, privacyniveau)
- `relationships`: getypeerde verbindingen naar gerelateerde pagina's (doel, soort, gewicht, betrouwbaarheid, bewijssoort, privacyniveau, notitie)

Begin voor een personenwiki met `reports/person-agent-directory.md` en open daarna de
persoonspagina met `wiki_get` voordat je contactgegevens of afgeleide
feiten gebruikt.

<Accordion title="Voorbeeld van een entiteitspagina">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - Routering binnen het voorbeeldecosysteem
notEnoughFor:
  - juridische goedkeuring
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: Voorbeeldecosysteem
  askFor:
    - Vragen over de voorbeelduitrol
  avoidAskingFor:
    - niet-gerelateerde factureringsbeslissingen
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: Andere persoon
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex is nuttig voor routering binnen het voorbeeldecosysteem.
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## Samenstelpijplijn

Bij het samenstellen worden wikipagina's gelezen, samenvattingen genormaliseerd
en wordt een machinegerichte momentopname opgeslagen in de gedeelde
SQLite-pluginstatus van OpenClaw. Runtimecode gebruikt de door de levenscyclus
beheerde eigenaarsmomentopname om SQLite te laden tijdens asynchrone
promptvoorbereiding; synchrone promptopbouw doorzoekt nooit Markdown en leest
geen cachebestanden. Samengestelde uitvoer ondersteunt ook de eerste
wiki-indexering voor zoeken/ophalen, het terugkoppelen van claim-ID's naar de
bijbehorende pagina's, compacte promptaanvullingen en rapportgeneratie.

Bronbewerkingen en herstelbewerkingen van de kluis worden pas na de volgende
samenstelling machinegericht beschikbaar. Bij het herstarten of vernieuwen van
de levenscyclus van de plugin wordt de causaal gekoppelde
samenstellingspublicatie van de kluis vergeleken met SQLite en wordt een
momentopname uit een nieuwere, teruggedraaide status geweigerd. Een compiler
die vóór het terugdraaien is gestart, kan niet publiceren ten opzichte van de
herstelde voorganger. Promptvoorbereiding peilt de kluis niet en installeert
geen bestandswatchers.
Na quarantaine vanwege een terugdraaiing wist een samenstelling in het actieve
proces de eigenaar onmiddellijk; een afzonderlijk compilerproces vereist
vernieuwing van de pluginlevenscyclus, zodat de daemon de nieuwe duurzame
publicatie kan bevestigen.
Samengestelde caches kunnen opnieuw worden opgebouwd: cacherijen van vóór
publicatie-epochs worden als missers behandeld en door de volgende
samenstelling vervangen; ze worden niet gemigreerd.

## Dashboards en statusrapporten

Wanneer `render.createDashboards` is ingeschakeld, onderhoudt de samenstelling
dashboards onder `reports/`:

| Rapport                             | Houdt bij                                          |
| ----------------------------------- | -------------------------------------------------- |
| `reports/open-questions.md`         | pagina's met onopgeloste vragen                    |
| `reports/contradictions.md`         | clusters van notities over tegenstrijdigheden      |
| `reports/low-confidence.md`         | pagina's en claims met lage betrouwbaarheid        |
| `reports/claim-health.md`           | claims zonder gestructureerd bewijs                |
| `reports/stale-pages.md`            | verouderde of onbekende actualiteit                |
| `reports/person-agent-directory.md` | routeringskaarten voor personen/entiteiten         |
| `reports/relationship-graph.md`     | gestructureerde relatieverbindingen                |
| `reports/provenance-coverage.md`    | dekking per bewijsklasse                           |
| `reports/privacy-review.md`         | niet-openbare privacyniveaus die vóór gebruik moeten worden beoordeeld |

## Zoeken en ophalen

Twee zoekbackends:

- `shared`: gebruik indien beschikbaar de gedeelde geheugenzoekstroom
- `local`: doorzoek de wiki lokaal

Drie corpora: `wiki`, `memory`, `all`.

- `wiki_search` / `wiki_get` gebruiken waar mogelijk samengestelde samenvattingen als eerste zoekronde
- claim-ID's verwijzen terug naar de bijbehorende pagina
- betwiste/verouderde/actuele claims beïnvloeden de rangschikking
- herkomstlabels blijven in de resultaten behouden

Zoekmodi (parameter `--mode` / tool `mode`):

| Modus             | Versterkt                                                      |
| ----------------- | -------------------------------------------------------------- |
| `auto`            | evenwichtige standaardinstelling                               |
| `find-person`     | persoonsachtige entiteiten, aliassen, gebruikersnamen, sociale profielen, canonieke ID's |
| `route-question`  | agentkaarten, hints voor waarvoor te vragen/waarvoor het meest geschikt, relatiecontext |
| `source-evidence` | bronpagina's en gestructureerde metagegevens voor bewijs       |
| `raw-claim`       | overeenkomende gestructureerde claims; retourneert metagegevens van claims/bewijs |

Wanneer een resultaat overeenkomt met een gestructureerde claim, retourneert `wiki_search`
`matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`,
`evidenceKinds` en `evidenceSourceIds` in de detailpayload. Tekstuitvoer
bevat compacte regels voor `Claim:` en `Evidence:` wanneer beschikbaar.

## Agenttools

| Tool          | Doel                                                                                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | huidige kluismodus en huidig bereik, herleide agent, status, beschikbaarheid van de Obsidian-CLI                                                              |
| `wiki_search` | wikipagina's doorzoeken en, indien geconfigureerd, het gedeelde geheugencorpus; accepteert `mode` voor het opzoeken van personen, routeren van vragen, bronbewijs of gedetailleerde inspectie van onbewerkte claims |
| `wiki_get`    | een wikipagina lezen op ID/pad, met terugval op het gedeelde geheugencorpus wanneer gedeeld zoeken is ingeschakeld en de zoekopdracht niets oplevert           |
| `wiki_apply`  | gerichte wijzigingen aan synthese/metagegevens zonder vrije paginabewerking                                                                                   |
| `wiki_lint`   | structurele controles, hiaten in herkomstgegevens, tegenstrijdigheden, openstaande vragen                                                                     |

De Plugin registreert ook een niet-exclusieve aanvulling op het geheugencorpus, zodat gedeelde
`memory_search` en `memory_get` de wiki kunnen bereiken wanneer de actieve geheugenplugin
corpusselectie ondersteunt.

## Gedrag van prompts en context

Wanneer `context.includeCompiledDigestPrompt` is ingeschakeld, voegen geheugenpromptsecties
een compacte, gecompileerde momentopname uit de pluginstatus toe: alleen toppagina's,
alleen topclaims, aantal tegenstrijdigheden, aantal vragen en kwalificaties voor
betrouwbaarheid/actualiteit. Dit is opt-in omdat het de promptvorm wijzigt; het is vooral relevant
voor contextengines of promptsamenstelling die expliciet geheugenaanvullingen
gebruiken.

## Configuratie

Plaats de configuratie onder `plugins.entries.memory-wiki.config`:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

Belangrijkste schakelaars:

| Sleutel                                    | Waarden / standaardwaarde                       | Opmerkingen                                                                   |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated` (standaard), `bridge`, `unsafe-local` | bepaalt invoer- en integratiegedrag                                            |
| `vault.scope`                              | `global` (standaard), `agent`                    | één gedeelde kluis of één onderliggende kluis per agent                       |
| `vault.path`                               | globale standaardwaarde `~/.openclaw/wiki/main` | exacte kluis voor globaal bereik; bovenliggende map voor agentbereik is standaard `~/.openclaw/wiki` |
| `vault.renderMode`                         | `native` (standaard), `obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | standaard `true`                                 | openbare artefacten van de actieve geheugenplugin importeren                  |
| `bridge.followMemoryEvents`                | standaard `true`                                 | gebeurtenislogboeken opnemen in brugmodus                                     |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | standaard `false`                                | vereist om `unsafe-local`-imports uit te voeren                           |
| `unsafeLocal.paths`                        | standaard `[]`                                   | expliciete lokale paden om te importeren in de modus `unsafe-local`       |
| `search.backend`                           | `shared` (standaard), `local`                    |                                                                               |
| `search.corpus`                            | `wiki` (standaard), `memory`, `all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | standaard `false`                                | compacte samenvattingsmomentopname van de geselecteerde agent toevoegen aan geheugenpromptsecties |
| `render.createBacklinks`                   | standaard `true`                                 | deterministische gerelateerde blokken genereren                               |
| `render.createDashboards`                  | standaard `true`                                 | dashboardpagina's genereren                                                   |

### Kluizen per agent

Stel `vault.scope` in op `agent` om elke geconfigureerde agent een afzonderlijke wiki te geven.
In dit bereik is `vault.path` een bovenliggende map en voegt OpenClaw de
genormaliseerde agent-ID toe:

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

Dit wordt herleid tot `~/.openclaw/wiki/support` en
`~/.openclaw/wiki/marketing`. Als `vault.path` in agentbereik wordt weggelaten, is de
bovenliggende map standaard `~/.openclaw/wiki`. De standaardagent `main` behoudt daardoor
het bestaande pad `~/.openclaw/wiki/main`.

Agenttools, gecompileerde promptoverzichten en de wiki-aanvulling die via
`memory_search` / `memory_get` beschikbaar wordt gesteld, herleiden de kluis vanuit de actieve agentcontext.
Geef bij CLI- en Gateway-aanroepen in een configuratie met meerdere geconfigureerde agents
de agent expliciet op met `openclaw wiki --agent <agentId> ...` of met
`agentId` van het Gateway-verzoek. Eén geconfigureerde agent blijft de standaard wanneer geen ID
wordt opgegeven.

In brugmodus accepteren imports met agentbereik een openbaar geheugenartefact alleen wanneer
`agentIds` daarvan de geselecteerde agent bevat. Artefacten die eigendom zijn van een andere agent,
geen eigendomsmetagegevens hebben of een onbekende eigenaar hebben, worden overgeslagen. Globaal bereik
behoudt het bestaande gedrag voor gedeelde artefacten.

<Warning>
Het wijzigen van `vault.scope` kopieert of splitst een bestaande kluis niet. In agentbereik
wordt een expliciet geconfigureerde `vault.path` een bovenliggende map; verplaats of
importeer bestaande pagina's daarom bewust voordat je productieagents omschakelt. Maak eerst een
back-up van de kluis.

Kluizen per agent vormen een kennisgrens binnen hetzelfde proces, geen beveiligingsgrens van het
besturingssysteem. Plugins en niet-gesandboxte tools met toegang tot het hostbestandssysteem kunnen
nog steeds de map van een andere agent lezen. Gebruik [sandboxing](/nl/gateway/sandboxing) of
[afzonderlijke Gateway-profielen](/nl/gateway/multiple-gateways) wanneer agents elkaar niet
vertrouwen.
</Warning>

### Voorbeeld: QMD + brugmodus

Gebruik dit wanneer je QMD wilt gebruiken voor het terughalen van informatie en `memory-wiki` voor een onderhouden
kennislaag. Elke laag blijft gericht: QMD houdt onbewerkte notities, sessie-
exports en extra verzamelingen doorzoekbaar, terwijl `memory-wiki`
stabiele entiteiten, claims, dashboards en bronpagina's compileert.

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

Hierdoor blijft QMD verantwoordelijk voor het terughalen uit het actieve geheugen, blijft `memory-wiki` gericht op
gecompileerde pagina's en dashboards en blijft de promptvorm ongewijzigd totdat je
bewust gecompileerde overzichtsprompts inschakelt.

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

Zie [CLI: wiki](/nl/cli/wiki) voor de volledige opdrachtreferentie, inclusief
`wiki okf import`, `wiki apply metadata`, `wiki unsafe-local import`,
`wiki chatgpt import` / `wiki chatgpt rollback` en de volledige reeks `wiki obsidian`-
subopdrachten.

## Ondersteuning voor Obsidian

Wanneer `vault.renderMode` is ingesteld op `obsidian`, schrijft de Plugin Markdown
dat geschikt is voor Obsidian en kan deze optioneel de officiële `obsidian`-CLI gebruiken voor
statuscontroles, zoeken in de kluis, het openen van een pagina, het aanroepen van een opdracht en het navigeren naar de
dagelijkse notitie. Dit is optioneel; de wiki werkt zonder Obsidian nog steeds in de
native modus.

Kluizen met agentbereik kunnen nog steeds Markdown gebruiken dat geschikt is voor Obsidian, maar de configuratie-
validatie wijst `obsidian.useOfficialCli: true` met `vault.scope: "agent"` af.
De huidige instelling `obsidian.vaultName` is globaal en kan niet voor elke agent een afzonderlijke
Obsidian-kluis selecteren. Gebruik in plaats daarvan de wikitools en CLI-bewerkingen,
of houd een door Obsidian beheerde wiki binnen globaal bereik.

## Aanbevolen workflow

<Steps>
<Step title="Behoud de Active Memory-plugin voor herinneringen">
Herinneringen, promotie en Dreaming blijven onder beheer van de geconfigureerde geheugenbackend.
</Step>
<Step title="Schakel memory-wiki in">
Begin met de modus `isolated`, tenzij je expliciet de bridge-modus wilt.
</Step>
<Step title="Gebruik wiki_search / wiki_get wanneer herkomst van belang is">
Geef hieraan de voorkeur boven `memory_search` wanneer je wiki-specifieke rangschikking of een geloofsstructuur op paginaniveau wilt.
</Step>
<Step title="Gebruik wiki_apply voor beperkte syntheses of metadata-updates">
Vermijd het handmatig bewerken van beheerde, gegenereerde blokken.
</Step>
<Step title="Voer wiki_lint uit na betekenisvolle wijzigingen">
Detecteert tegenstrijdigheden, open vragen en hiaten in de herkomst.
</Step>
<Step title="Schakel dashboards in voor inzicht in verouderde gegevens en tegenstrijdigheden">
Stel `render.createDashboards: true` in (standaard).
</Step>
</Steps>

## Gerelateerde documentatie

- [Overzicht van geheugen](/nl/concepts/memory)
- [CLI: geheugen](/nl/cli/memory)
- [CLI: wiki](/nl/cli/wiki)
- [Overzicht van de Plugin-SDK](/nl/plugins/sdk-overview)
