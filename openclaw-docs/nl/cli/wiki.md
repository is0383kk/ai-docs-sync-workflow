---
read_when:
    - Je wilt de memory-wiki-CLI gebruiken
    - Je documenteert of wijzigt `openclaw wiki`
summary: CLI-referentie voor `openclaw wiki` (status, zoeken, compileren, linten, toepassen en bridge voor de memory-wiki-kluis, ChatGPT-import en Obsidian-hulpmiddelen)
title: Wiki
x-i18n:
    generated_at: "2026-07-27T06:10:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

Inspecteer en onderhoud de `memory-wiki`-kluis. Wordt geleverd door de gebundelde optionele `memory-wiki`-Plugin. Schakel deze vóór het eerste gebruik in:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

Gerelateerd: [Memory Wiki-Plugin](/nl/plugins/memory-wiki), [Overzicht van geheugen](/nl/concepts/memory), [CLI: geheugen](/nl/cli/memory)

## Veelgebruikte opdrachten

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "wie kan ik over Teams raadplegen?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Samenvatting van Alpha" \
  --body "Korte synthesetekst" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Nog steeds actief?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Agentselectie

Wanneer `plugins.entries.memory-wiki.config.vault.scope` `agent` is, selecteer je de
kluis met de `--agent <id>`-optie op het hoogste niveau:

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "terugbetalingsbeleid"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

In een configuratie met meerdere geconfigureerde agents is `--agent` vereist voor CLI-
bewerkingen, zodat een opdracht niet zomaar een willekeurige standaardkluis kan lezen of schrijven. Als
er slechts één agent is geconfigureerd, blijft die agent de standaard. Onbekende agent-id's
leiden tot een fout voordat de kluisbewerking begint. De optie verandert het geselecteerde
pad niet wanneer `vault.scope` `global` is.

Gateway-clients volgen dezelfde regel: geef `agentId` door bij door een kluis ondersteunde `wiki.*`-
aanvragen in een agentgebonden configuratie met meerdere agents. Een ontbrekende of onbekende id is een
fout. Agentbeurten, wiki-tools, aanvullingen op de geheugencorpus en gecompileerde prompt-
samenvattingen bevatten al de actieve runtimecontext van de agent.

## Opdrachten

### `wiki status`

Toon de kluismodus en het bereik, de herleide agent, de status en de beschikbaarheid van de Obsidian-CLI. Gebruik dit eerst om te controleren of de bedoelde kluis is geïnitialiseerd, de bridge-modus correct werkt of de Obsidian-integratie beschikbaar is.

Wanneer de bridge-modus actief en geconfigureerd is om geheugenartefacten te lezen, vraagt deze opdracht de actieve Gateway op, zodat dezelfde context van de actieve geheugen-Plugin zichtbaar is als voor het agent-/runtimegeheugen.

### `wiki doctor`

Voer statuscontroles voor de wiki uit en rapporteer uitvoerbare oplossingen. Wordt afgesloten met een niet-nulwaarde wanneer de wiki niet in orde is.

Wanneer de bridge-modus actief en geconfigureerd is om geheugenartefacten te lezen, vraagt deze opdracht de actieve Gateway op voordat het rapport wordt samengesteld. Uitgeschakelde bridge-imports en bridge-configuraties die geen geheugenartefacten lezen, blijven lokaal/offline.

Veelvoorkomende problemen:

- bridge-modus ingeschakeld zonder openbare geheugenartefacten
- ongeldige of ontbrekende kluisindeling
- ontbrekende externe Obsidian-CLI wanneer de Obsidian-modus wordt verwacht

### `wiki init`

Maak de indeling van de wikikluis en beginpagina's aan, inclusief indexen op het hoogste niveau en cachemappen.

### `wiki ingest <path>`

Importeer een lokaal Markdown- of tekstbestand als bronpagina in de map `sources/` van de wiki. `<path>` moet een lokaal bestandspad zijn; importeren via een URL wordt momenteel niet ondersteund. Binaire bestanden worden geweigerd.

Geïmporteerde bronpagina's bevatten frontmatter over de herkomst (`sourceType: local-file`, `sourcePath`, `ingestedAt`). Na het importeren wordt de kluis altijd opnieuw gecompileerd.

Vlaggen: `--title <title>` overschrijft de brontitel (standaard: afgeleid van de bestandsnaam).

### `wiki okf import <path>`

Importeer een uitgepakt Open Knowledge Format-pakket in conceptpagina's van de wiki.

De importfunctie leest elk niet-gereserveerd `.md`-conceptdocument in de OKF-mappenstructuur, vereist een niet-leeg `type`-veld en behandelt onbekende OKF-waarden voor `type` als algemene concepten. Gereserveerde OKF-bestanden `index.md` en `log.md` worden niet als concepten geïmporteerd.

Geïmporteerde pagina's worden onder `concepts/` in één niveau geplaatst, zodat bestaande flows voor compileren, zoeken, ophalen, samenvattingen en dashboards van de wiki ze onmiddellijk kunnen gebruiken. De oorspronkelijke OKF-concept-id, `type`, `resource`, `tags`, het tijdstempel, het bronpad en de volledige frontmatter blijven behouden in de frontmatter van de pagina. Interne OKF-Markdown-links worden herschreven naar de gegenereerde wikipagina's; defecte of externe links blijven ongewijzigd. Na de import wordt de kluis altijd opnieuw gecompileerd.

Voorbeelden:

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery-tabel" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

Bouw indexen, gerelateerde blokken, dashboards en de gecompileerde momentopname voor query's/prompts opnieuw op. De momentopname wordt opgeslagen in de gedeelde SQLite-Pluginstatus van OpenClaw en in het geheugen gehouden voor synchrone promptprojectie; er worden geen cachebestanden in de kluis aangemaakt.

Als `render.createDashboards` is ingeschakeld, vernieuwt de compilatie ook rapportpagina's.

### `wiki lint`

Controleer de kluis en schrijf een rapport over:

- structurele problemen (defecte links, ontbrekende/dubbele id's, ontbrekend paginatype of ontbrekende titel, ongeldige frontmatter)
- hiaten in herkomstgegevens (ontbrekende bron-id's, ontbrekende importherkomst)
- tegenstrijdigheden (gemarkeerde tegenstrijdigheden, conflicterende beweringen)
- openstaande vragen
- pagina's en beweringen met lage betrouwbaarheid
- verouderde pagina's en beweringen

Voer dit uit na belangrijke wiki-updates.

### `wiki search <query>`

Doorzoek wiki-inhoud. Het gedrag is afhankelijk van de configuratie:

- `search.backend`: `shared` of `local`
- `search.corpus`: `wiki`, `memory` of `all`
- `--mode`: `auto`, `find-person`, `route-question`, `source-evidence` of `raw-claim`

Gebruik `wiki search` voor wikispecifieke rangschikking en herkomstgegevens. Voor één brede gedeelde ophaalactie geef je de voorkeur aan `openclaw memory search` wanneer de actieve geheugen-Plugin gedeeld zoeken aanbiedt.

Zoekmodi:

- `find-person`: aliassen, gebruikersnamen, sociale accounts, canonieke id's en persoonspagina's
- `route-question`: aanwijzingen voor wie je kunt raadplegen/waarvoor iemand het meest geschikt is en context over relaties
- `source-evidence`: bronpagina's en gestructureerde bewijsvelden
- `raw-claim`: gestructureerde beweringstekst met metadata voor beweringen/bewijs

Voorbeelden:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "wie kent de uitrol van Teams?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "sterke route Teams" --mode raw-claim --json
```

Tekstuitvoer bevat regels voor `Claim:` en `Evidence:` wanneer een resultaat overeenkomt met een gestructureerde bewering. JSON-uitvoer stelt daarnaast `matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`, `evidenceKinds` en `evidenceSourceIds` beschikbaar voor verdere analyse door de agent.

### `wiki get <lookup>`

Lees een wikipagina op id of relatief pad.

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

Pas gerichte wijzigingen toe zonder pagina's vrijelijk te bewerken:

- `apply synthesis <title>`: maak een synthesepagina met een beheerde samenvattingstekst of vernieuw deze
- `apply metadata <lookup>`: werk metadata van een bestaande pagina bij

Beide accepteren `--source-id`, `--contradiction`, `--question` (elk herhaalbaar), `--confidence <n>` (0-1) en `--status <status>`. `apply metadata` accepteert ook `--clear-confidence` om een opgeslagen betrouwbaarheidswaarde te verwijderen. Dit is de ondersteunde manier om wikipagina's verder te ontwikkelen, zodat beheerde gegenereerde blokken intact blijven.

### `wiki bridge import`

Importeer openbare geheugenartefacten van de actieve geheugen-Plugin in door de bridge ondersteunde bronpagina's. Gebruik dit in de modus `bridge` om de nieuwste geëxporteerde geheugenartefacten naar de wikikluis over te brengen.

Bij het lezen van actieve bridge-artefacten routeert de CLI de import via Gateway-RPC, zodat de context van de runtimegeheugen-Plugin wordt gebruikt. Als bridge-imports zijn uitgeschakeld of het lezen van artefacten is uitgeschakeld, behoudt de opdracht het lokale/offline gedrag waarbij niets wordt geïmporteerd. Het vernieuwen van de index na de import wordt bepaald door `ingest.autoCompile`.

### `wiki unsafe-local import`

Importeer uit expliciet geconfigureerde lokale paden (`unsafeLocal.paths`) in de modus `unsafe-local`. Dit is bewust experimenteel en uitsluitend bedoeld voor dezelfde machine. Het vernieuwen van de index na de import wordt bepaald door `ingest.autoCompile`.

### `wiki chatgpt import`

Importeer een ChatGPT-export in conceptbronpagina's van de wiki.

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| Vlag              | Standaard    | Beschrijving                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | (verplicht) | ChatGPT-exportmap of pad naar `conversations.json`.        |
| `--dry-run`       | `false`    | Bekijk aantallen aangemaakte/bijgewerkte/overgeslagen pagina's zonder pagina's te schrijven. |

Een import zonder testmodus die een pagina wijzigt, registreert een importuitvoerings-id. Deze wordt in de samenvatting weergegeven en is nodig om de import terug te draaien.

### `wiki chatgpt rollback <run-id>`

Draai een eerder toegepaste ChatGPT-importuitvoering terug, waarbij aangemaakte pagina's worden verwijderd en overschreven pagina's worden hersteld. Doet niets (en rapporteert `alreadyRolledBack`) als de uitvoering al is teruggedraaid.

### `wiki obsidian ...`

Obsidian-hulpopdrachten voor kluizen die in een Obsidian-vriendelijke modus worden uitgevoerd: `status`, `search`, `open`, `command`, `daily`. Hiervoor is de officiële `obsidian`-CLI vereist op `PATH` wanneer `obsidian.useOfficialCli` is ingeschakeld.

Configuratievalidatie weigert `obsidian.useOfficialCli: true` wanneer
`vault.scope` `agent` is, omdat `obsidian.vaultName` één algemene instelling is
en geen toewijzing per agent. Obsidian-vriendelijke Markdown-weergave blijft
beschikbaar.

## Praktische gebruiksrichtlijnen

- Gebruik `wiki search` + `wiki get` wanneer herkomstgegevens en pagina-identiteit belangrijk zijn.
- Gebruik `wiki apply` in plaats van beheerde gegenereerde secties handmatig te bewerken.
- Gebruik `wiki lint` voordat je tegenstrijdige inhoud of inhoud met een lage betrouwbaarheid vertrouwt.
- Gebruik `wiki compile` na bulkimports of bronwijzigingen wanneer je onmiddellijk vernieuwde dashboards en gecompileerde samenvattingen wilt.
- Gebruik `wiki okf import` wanneer een gegevenscatalogus, documentatie-export of verrijkingspijplijn voor agents al OKF-Markdown-pakketten produceert.
- Gebruik `wiki bridge import` wanneer de bridge-modus afhankelijk is van pas geëxporteerde geheugenartefacten.

## Gekoppelde configuratie

Het gedrag van `openclaw wiki` wordt bepaald door:

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

Zie [Memory Wiki-Plugin](/nl/plugins/memory-wiki) voor het volledige configuratiemodel.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Memory Wiki](/nl/plugins/memory-wiki)
