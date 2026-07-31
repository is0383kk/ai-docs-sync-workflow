---
read_when:
    - Je bent installaties van Plugin-pakketten aan het debuggen
    - Je wijzigt het opstartgedrag van de Plugin, doctor of de installatie via de pakketbeheerder
    - Je onderhoudt verpakte OpenClaw-installaties of meegeleverde pluginmanifesten
sidebarTitle: Dependencies
summary: Hoe OpenClaw pluginpakketten installeert en plugin-afhankelijkheden oplost
title: Resolutie van Plugin-afhankelijkheden
x-i18n:
    generated_at: "2026-07-27T05:06:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw verwerkt Plugin-afhankelijkheden alleen tijdens installatie/bijwerken. Tijdens het laden
voert de runtime nooit een pakketbeheerder uit, herstelt deze geen afhankelijkheidsstructuur en wijzigt
deze de OpenClaw-pakketmap niet.

## Verdeling van verantwoordelijkheden

Plugin-pakketten beheren hun eigen afhankelijkheidsgraaf:

- Runtime-afhankelijkheden staan in de `dependencies` of
  `optionalDependencies` van het Plugin-pakket.
- SDK-/core-imports zijn peer-imports of door OpenClaw geleverde imports.
- Plugins voor lokale ontwikkeling leveren hun eigen reeds geïnstalleerde afhankelijkheden.
- npm- en git-Plugins worden geïnstalleerd in pakketroots die door OpenClaw worden beheerd.

OpenClaw beheert alleen de levenscyclus van de Plugin:

- De bron van de Plugin ontdekken.
- Het pakket installeren of bijwerken wanneer daar expliciet om wordt gevraagd.
- Installatiemetagegevens vastleggen.
- Het ingangspunt van de Plugin laden.
- Mislukken met een bruikbare foutmelding wanneer afhankelijkheden ontbreken.

## Installatieroots

OpenClaw gebruikt stabiele roots per bron:

- npm-pakketten worden geïnstalleerd in projecten per Plugin onder
  `~/.openclaw/npm/projects/<encoded-package>`.
- git-pakketten worden gekloond onder `~/.openclaw/git`.
- Lokale installaties en installaties vanuit paden/archieven worden zonder herstel van
  afhankelijkheden gekopieerd of gebruikt als verwijzing.

npm-installaties worden in die projectroot per Plugin uitgevoerd met:

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` gebruikt dezelfde npm-projectroot per Plugin
voor een lokaal npm-pack-tararchief: OpenClaw leest de npm-metagegevens van het
tararchief, voegt het aan het beheerde project toe als een gekopieerde `file:`-afhankelijkheid, voert
de normale npm-installatie hierboven uit en verifieert vervolgens de geïnstalleerde lockfile-metagegevens
voordat de Plugin wordt vertrouwd. Dit pad bestaat voor pakketacceptatie en
bewijs voor releasekandidaten, waarbij een lokaal pakketartefact zich moet gedragen als het
registerartefact dat het simuleert.

Gebruik `npm-pack:` bij het testen van officiële of externe Plugin-pakketten vóór
publicatie. Een onbewerkt archief of een installatie vanuit een pad is nuttig voor lokale foutopsporing, maar
bewijst niet hetzelfde afhankelijkheidspad als een geïnstalleerd npm- of ClawHub-
pakket. `npm-pack:` bewijst de beheerde pakketinstallatievorm; op zichzelf
bewijst het niet dat de Plugin officieel aan de catalogus gekoppelde inhoud is.

Wanneer gedrag afhankelijk is van de status van een gebundelde Plugin of een vertrouwde officiële Plugin,
combineer je het lokale pakketbewijs met een officiële installatie vanuit de catalogus of een
gepubliceerd pakketpad waarin officieel vertrouwen wordt vastgelegd. Toegang tot bevoorrechte helpers
en verwerking van het vertrouwde officiële bereik moeten op dat vertrouwde
installatiepad worden gevalideerd en niet uit een lokale tararchiefinstallatie worden afgeleid.

Als een Plugin tijdens runtime mislukt vanwege een ontbrekende import, herstel dan het pakketmanifest
in plaats van het beheerde project handmatig te repareren. Runtime-imports horen in
de `dependencies` of `optionalDependencies` van het Plugin-pakket; `devDependencies`
worden niet geïnstalleerd voor beheerde runtimeprojecten. Een lokale `npm install` in
`~/.openclaw/npm/projects/<encoded-package>` kan een tijdelijke
diagnose mogelijk maken, maar geldt niet als bewijs voor pakketacceptatie omdat de volgende installatie of
bijwerking het project opnieuw aanmaakt op basis van pakketmetagegevens.

npm kan transitieve afhankelijkheden hijsen naar de `node_modules` van het project per Plugin,
naast het Plugin-pakket. OpenClaw scant de beheerde projectroot
voordat de installatie wordt vertrouwd en verwijdert dat project bij de-installatie, zodat
gehesen runtime-afhankelijkheden binnen de opschoningsgrens van die Plugin blijven.

Gepubliceerde npm-Plugin-pakketten kunnen `npm-shrinkwrap.json` meeleveren; npm gebruikt dat
publiceerbare lockbestand tijdens de installatie en de beheerde npm-projectroot van OpenClaw
ondersteunt dit via het normale installatiepad. Publiceerbare
Plugin-pakketten die eigendom zijn van OpenClaw moeten een pakketspecifieke shrinkwrap bevatten die is gegenereerd op basis van
de gepubliceerde afhankelijkheidsgraaf van dat pakket:

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

De generator verwijdert Plugin-`devDependencies`, past het beleid voor workspace-overschrijvingen
toe en schrijft `extensions/<id>/npm-shrinkwrap.json` voor elke Plugin met
`openclaw.release.publishToNpm: true`. Plugin-pakketten van derden mogen ook
een shrinkwrap meeleveren; OpenClaw vereist er geen voor communitypakketten, maar
npm respecteert deze wanneer aanwezig.

Inspecteer voordat je een lokaal pakket als bewijs voor een releasekandidaat beschouwt het
tararchief dat wordt geïnstalleerd:

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

Controleer bij wijzigingen aan afhankelijkheden ook of een productie-installatie de
runtimepakketten zonder ontwikkelingsafhankelijkheden kan vinden:

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

npm-Plugin-pakketten die eigendom zijn van OpenClaw kunnen ook worden gepubliceerd met expliciete
`bundledDependencies`. Het npm-publicatiepad legt de lijst met namen van runtime-afhankelijkheden
eroverheen, verwijdert workspace-metagegevens die alleen voor ontwikkeling zijn bedoeld uit het gepubliceerde manifest,
voert een npm-installatie zonder scripts uit voor de pakketspecifieke runtime-afhankelijkheden
en verpakt of publiceert vervolgens het Plugin-tararchief met die afhankelijkheidsbestanden
erin. Pakketten met veel native onderdelen (Codex, ACPX, Copilot, llama.cpp,
memory-lancedb, Tlon) kiezen hiervoor niet met
`openclaw.release.bundleRuntimeDependencies: false`; ze leveren nog steeds een
shrinkwrap mee, maar npm lost runtime-afhankelijkheden tijdens de installatie op in plaats van
elk platformbinair bestand in het Plugin-tararchief in te sluiten. Het rootpakket `openclaw`
bundelt niet zijn volledige afhankelijkheidsstructuur.

Plugins die `openclaw/plugin-sdk/*` importeren, declareren `openclaw` als peer-
afhankelijkheid. OpenClaw staat niet toe dat npm een afzonderlijke registerkopie van het
hostpakket in een beheerd project installeert, omdat een verouderd hostpakket
de peer-resolutie van npm binnen die Plugin kan beïnvloeden. Beheerde npm-installaties slaan de peer-
resolutie/materialisatie van npm over en OpenClaw stelt na installatie of bijwerking opnieuw Plugin-lokale
`node_modules/openclaw`-koppelingen in voor geïnstalleerde pakketten die de host-
peer declareren.

git-installaties klonen of vernieuwen de repository en voeren vervolgens het volgende uit:

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

De geïnstalleerde Plugin wordt vervolgens vanuit die pakketmap geladen, zodat
de resolutie van pakketspecifieke en bovenliggende `node_modules` op dezelfde manier werkt
als bij een normaal Node-pakket.

## Lokale Plugins

Lokale Plugins zijn door ontwikkelaars beheerde mappen. OpenClaw voert nooit
`npm install`, `pnpm install` of herstel van afhankelijkheden voor deze Plugins uit; als een lokale
Plugin afhankelijkheden heeft, installeer je die in de Plugin voordat je deze laadt.

Lokale TypeScript-Plugins van derden worden als noodoplossing via Jiti geladen.
Verpakte JavaScript-Plugins en gebundelde interne Plugins worden in plaats daarvan geladen via native
import/require.

## Opstarten en herladen

Bij het opstarten van de Gateway en het herladen van de configuratie worden nooit Plugin-afhankelijkheden geïnstalleerd. Deze processen
lezen de installatiegegevens van de Plugin, berekenen het ingangspunt en laden het.

Een ontbrekende afhankelijkheid tijdens runtime zorgt ervoor dat het laden van de Plugin mislukt met een foutmelding die
de beheerder naar een expliciete oplossing verwijst:

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` ruimt verouderde, door OpenClaw gegenereerde afhankelijkheidsstatus op en kan
downloadbare Plugins herstellen die ontbreken in lokale installatiegegevens wanneer
de configuratie er nog steeds naar verwijst. Doctor herstelt geen afhankelijkheden voor een
reeds geïnstalleerde lokale Plugin.

## Gebundelde Plugins

Lichte en voor de core kritieke gebundelde Plugins worden als onderdeel van OpenClaw geleverd. Ze
mogen geen zware runtime-afhankelijkheidsstructuur bevatten, of moeten worden verplaatst naar een
downloadbaar pakket op ClawHub/npm.

Zie voor de huidige gegenereerde lijst met Plugins die in het corepakket worden geleverd,
extern worden geïnstalleerd of alleen als broncode beschikbaar blijven:
[Plugin-inventaris](/nl/plugins/plugin-inventory).

Manifesten van gebundelde Plugins mogen niet om het klaarzetten van afhankelijkheden vragen. Grote of
optionele Plugin-functionaliteit moet als een normale Plugin worden verpakt en
via hetzelfde npm-/git-/ClawHub-pad als Plugins van derden worden geïnstalleerd.

In broncodecheck-outs behandelt OpenClaw de repository als een pnpm-monorepo.
Na `pnpm install` worden gebundelde Plugins geladen vanuit `extensions/<id>`, zodat
pakketspecifieke workspace-afhankelijkheden beschikbaar zijn en wijzigingen direct worden
overgenomen. Ontwikkeling vanuit een broncodecheck-out ondersteunt alleen pnpm; een gewone `npm install` in
de repositoryroot bereidt de afhankelijkheden van gebundelde Plugins niet voor.

| Installatievorm                  | Locatie van gebundelde Plugin         | Eigenaar van afhankelijkheden                                          |
| -------------------------------- | ------------------------------------- | ---------------------------------------------------------------------- |
| `npm install -g openclaw`        | Gebouwde runtimestructuur in het pakket | OpenClaw-pakket en expliciete installatie-/bijwerk-/doctorflows voor Plugins |
| Git-check-out plus `pnpm install` | `extensions/<id>`-workspacepakketten  | De pnpm-workspace, inclusief de eigen afhankelijkheden van elk Plugin-pakket |
| `openclaw plugins install ...`   | Beheerde npm-project-/git-/ClawHub-root | De installatie-/bijwerkflow van de Plugin                              |

## Opschoning van verouderde gegevens

Oudere versies van OpenClaw genereerden bij het opstarten
of tijdens herstel door Doctor afhankelijkheidsroots voor gebundelde Plugins. De huidige Doctor-opschoning verwijdert die verouderde
mappen en symbolische koppelingen met `--fix`, waaronder oude `plugin-runtime-deps`-
roots, globale symbolische pakketkoppelingen voor Node-prefixen die verwijzen naar opgeschoonde
`plugin-runtime-deps`-doelen, `.openclaw-runtime-deps*`-manifesten, gegenereerde
Plugin-`node_modules`, installatiefasemappen en pakketspecifieke pnpm-
stores. De verpakte postinstallatie verwijdert die globale symbolische koppelingen ook voordat
de verouderde doelroots worden opgeschoond, zodat upgrades geen loshangende ESM-
pakketimports achterlaten.

Oudere npm-installaties gebruikten ook een gedeelde `~/.openclaw/npm/node_modules`-root.
De huidige installatie-, bijwerk-, de-installatie- en Doctor-flows herkennen die
verouderde platte root nog steeds, maar alleen voor herstel en opschoning. Nieuwe npm-installaties maken
in plaats daarvan projectroots per Plugin aan.
