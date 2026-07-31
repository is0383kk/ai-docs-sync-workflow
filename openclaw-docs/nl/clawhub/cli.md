---
read_when:
    - De ClawHub-CLI gebruiken
    - Installatie-, update- of publicatieproblemen oplossen
summary: 'CLI-referentie: opdrachten, vlaggen, configuratie en lockfile-gedrag.'
x-i18n:
    generated_at: "2026-07-27T05:38:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

CLI-pakket: `clawhub`, binair bestand: `clawhub`.

Installeer het globaal met npm of pnpm:

```bash
npm i -g clawhub
# of
pnpm add -g clawhub
```

Verifieer het vervolgens:

```bash
clawhub --help
clawhub login
clawhub whoami
```

## Globale vlaggen

- `--workdir <dir>`: werkmap (standaard: cwd; valt terug op de Clawdbot-werkruimte indien geconfigureerd)
- `--dir <dir>`: installatiemap onder de werkmap (standaard: `skills`)
- `--site <url>`: basis-URL voor aanmelden via de browser (standaard: `https://clawhub.ai`)
- `--registry <url>`: API-basis-URL (standaard: gedetecteerd, anders `https://clawhub.ai`)
- `--no-input`: prompts uitschakelen

Overeenkomstige omgevingsvariabelen:

- `CLAWHUB_SITE` (verouderd: `CLAWDHUB_SITE`)
- `CLAWHUB_REGISTRY` (verouderd: `CLAWDHUB_REGISTRY`)
- `CLAWHUB_WORKDIR` (verouderd: `CLAWDHUB_WORKDIR`)

### HTTP-proxy

De CLI respecteert standaardomgevingsvariabelen voor HTTP-proxy's voor systemen achter
bedrijfsproxy's of beperkte netwerken:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

Wanneer een van deze variabelen is ingesteld, leidt de CLI uitgaande verzoeken via
de opgegeven proxy. `HTTPS_PROXY` wordt gebruikt voor HTTPS-verzoeken, `HTTP_PROXY`
voor gewone HTTP-verzoeken. `NO_PROXY` / `no_proxy` wordt gerespecteerd om de proxy voor
specifieke hosts of domeinen te omzeilen.

Dit is vereist op systemen waar directe uitgaande verbindingen worden geblokkeerd
(bijvoorbeeld Docker-containers, Hetzner-VPS'en met uitsluitend internet via een proxy,
bedrijfsfirewalls).

Voorbeeld:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "mijn zoekopdracht"
```

Wanneer geen proxyvariabele is ingesteld, blijft het gedrag ongewijzigd (directe verbindingen).

## Configuratiebestand

Slaat je API-token en de gecachte register-URL op.

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` of `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- Verouderde terugvaloptie: als `clawhub/config.json` nog niet bestaat, maar `clawdhub/config.json` wel, gebruikt de CLI het verouderde pad opnieuw
- overschrijven: `CLAWHUB_CONFIG_PATH` (verouderd: `CLAWDHUB_CONFIG_PATH`)

## Opdrachten

### `login` / `auth login`

- Standaard: opent de browser op `<site>/cli/auth` en voltooit het proces via een loopback-callback.
- Headless: `clawhub login --token clh_...`
- Interactief op afstand/headless: `clawhub login --device` drukt een code af en wacht terwijl je deze autoriseert op `<site>/cli/device`.

### `whoami`

- Verifieert het opgeslagen token via `/api/v1/whoami`.

### `token`

- Drukt het opgeslagen API-token af naar stdout.
- Nuttig om een lokaal aanmeldingstoken via een pipe door te geven aan opdrachten voor het instellen van CI-geheimen.

### `star <skill>` / `unstar <skill>`

- Voegt een skill toe aan of verwijdert deze uit je bladwijzers. Opdrachtnamen blijven `star` en
  `unstar` voor compatibiliteit.
- Roept `POST /api/v1/stars/<slug>` en `DELETE /api/v1/stars/<slug>` aan.
- `--yes` slaat de bevestiging over.

### `search <query...>`

- Roept `/api/v1/search?q=...` aan.
- De uitvoer bevat de slug van de skill, de gebruikersnaam van de eigenaar, de weergavenaam en de relevantiescore.
- Bij het zoeken krijgen exacte overeenkomsten met tokens in de slug/naam voorrang op downloadpopulariteit. Een zelfstandig slugtoken zoals `map` komt sterker overeen met `personal-map` dan met de subtekenreeks in `amap`.
- Populariteit is een kleine voorafgaande rangschikkingsfactor, geen garantie op de hoogste positie.
- Als een skill zou moeten verschijnen maar dat niet doet, voer dan aangemeld `clawhub inspect @owner/slug` uit om voor de eigenaar zichtbare moderatiediagnostiek te controleren voordat je metagegevens hernoemt.

### `explore`

- Geeft de nieuwste skills weer via `/api/v1/skills?limit=...&sort=createdAt` (aflopend gesorteerd op `createdAt`).
- Vlaggen:
  - `--limit <n>` (1-200, standaard: 25)
  - `--sort newest|updated|rating|downloads|trending` (standaard: nieuwste). Verouderde aliassen voor installatiesortering blijven werken voor compatibiliteit.
  - `--json` (machineleesbare uitvoer)
- Uitvoer: `<slug>  v<version>  <age>  <summary>` (samenvatting afgekapt tot 50 tekens).

### `inspect @owner/slug`

- Haalt metagegevens en versiebestanden van de skill op zonder deze te installeren.
- `--version <version>`: een specifieke versie inspecteren (standaard: nieuwste).
- `--tag <tag>`: een getagde versie inspecteren (bijvoorbeeld `latest`).
- `--versions`: versiegeschiedenis weergeven (eerste pagina).
- `--limit <n>`: maximaal aantal weer te geven versies (1-200).
- `--files`: bestanden voor de geselecteerde versie weergeven.
- `--file <path>`: onbewerkte bestandsbytes ophalen (limiet van 10MB).
- `--json`: machineleesbare uitvoer; `--file` bevat de exacte bytes als base64 en, indien beschikbaar, UTF-8-tekst.

### `install @owner/slug`

- Bepaalt de nieuwste versie voor de genoemde eigenaar en skill.
- Downloadt een zipbestand via `/api/v1/download`.
- Pakt het uit in `<workdir>/<dir>/<slug>`.
- Weigert vastgezette skills te overschrijven; voer eerst `clawhub unpin <skill>` uit.
- Schrijft:
  - `<workdir>/.clawhub/lock.json` (verouderd: `.clawdhub`)
  - `<skill>/.clawhub/origin.json` (verouderd: `.clawdhub`)

### `uninstall <skill>`

- Verwijdert `<workdir>/<dir>/<slug>` en verwijdert de vermelding uit het vergrendelingsbestand.
- Verstuurt naar beste vermogen telemetrie wanneer je bent aangemeld, zodat de huidige installatieaantallen kunnen worden
  gedeactiveerd.
- Interactief: vraagt om bevestiging.
- Niet-interactief (`--no-input`): vereist `--yes`.

### `list`

- Leest `<workdir>/.clawhub/lock.json` (verouderd: `.clawdhub`).
- Toont `pinned` naast skills die met `clawhub pin` zijn bevroren, inclusief de optionele reden.

### `pin <skill>`

- Markeert een geïnstalleerde skill als vastgezet in het vergrendelingsbestand.
- `--reason <text>` legt vast waarom de skill is bevroren.
- Vastgezette skills worden overgeslagen door `update --all` en geweigerd door een directe `update <skill>`.
- Vastgezette skills weigeren ook `install --force`, zodat de lokale bytes niet per ongeluk kunnen worden vervangen.

### `unpin <skill>`

- Verwijdert de vastzetting uit het vergrendelingsbestand van een geïnstalleerde skill, zodat toekomstige updates deze kunnen wijzigen.

### `update [@owner/slug]` / `update --all`

- Berekent de vingerafdruk op basis van lokale bestanden.
- Als de vingerafdruk overeenkomt met een bekende versie: geen prompt.
- Als de vingerafdruk niet overeenkomt:
  - wordt dit standaard geweigerd
  - wordt overschreven met `--force` (of na een prompt, indien interactief)
- Vastgezette skills worden nooit bijgewerkt door `--force`.
- `update <skill>` mislukt onmiddellijk voor vastgezette skills en vraagt je eerst `clawhub unpin <skill>` uit te voeren.
- `update --all` slaat vastgezette slugs over en drukt een samenvatting af van wat bevroren bleef.

### `skill publish <path>`

- Vergelijkt de vingerafdruk van de lokale bundel met ClawHub en wordt succesvol afgesloten wanneer
  de inhoud al is gepubliceerd.
- Nieuwe skills gebruiken standaard `1.0.0`; gewijzigde skills gebruiken standaard de volgende patchversie.
- `--version <version>` selecteert expliciet een versie en publiceert zelfs wanneer de
  inhoud overeenkomt met een bestaande versie.
- `--dry-run` bepaalt het publicatieresultaat zonder te uploaden; `--json` drukt een
  machineleesbaar resultaat af.
- `--owner <handle>` publiceert onder de uitgeversnaam van een organisatie/gebruiker wanneer de
  actor uitgeverstoegang heeft.
- `--migrate-owner` verplaatst een bestaande skill naar `--owner` terwijl een nieuwe
  versie wordt gepubliceerd. Vereist beheerders-/eigenaarstoegang voor beide uitgevers.
- Het gedrag voor eigenaren en beoordelingen wordt uitgelegd in `docs/publishing.md`.
- Het publiceren van een skill betekent dat deze onder `MIT-0` op ClawHub wordt uitgebracht.
- Gepubliceerde skills mogen zonder naamsvermelding vrij worden gebruikt, gewijzigd en verspreid.
- ClawHub ondersteunt geen betaalde skills of prijzen per skill.
- Verouderde alias: `publish <path>`.

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

De herbruikbare workflow
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
van ClawHub roept `skill publish` aan voor één `skill_path`, of voor elke directe skillmap
onder `root` (standaard: `skills`). Ongewijzigde skills worden overgeslagen en hetzelfde
automatische gedrag voor patchversies wordt gebruikt.

Stel `dry_run: true` in om een voorbeeld te bekijken zonder token. Voor daadwerkelijke publicaties is het
geheim `clawhub_token` vereist.

### `sync`

- Scant de huidige werkmap, de geconfigureerde skillsmap en alle
  `--root <dir>`-mappen op lokale skillmappen die `SKILL.md` of
  `skill.md` bevatten.
- Vergelijkt de vingerafdruk van elke lokale skill met ClawHub en publiceert alleen nieuwe of
  gewijzigde skills.
- Nieuwe skills worden gepubliceerd als `1.0.0`; gewijzigde skills worden standaard als de volgende patchversie
  gepubliceerd. Gebruik `--bump minor|major` voor updatebatches die met een
  grotere semver-stap moeten worden verhoogd.
- `--dry-run` toont het publicatieplan zonder te uploaden; `--json` drukt een
  machineleesbaar plan af.
- `--all` publiceert elke nieuwe of gewijzigde skill zonder prompt. Zonder
  `--all` kun je in interactieve terminals de te publiceren skills selecteren.
- `--owner <handle>` publiceert onder de uitgeversnaam van een organisatie/gebruiker wanneer de
  actor uitgeverstoegang heeft.
- `sync` publiceert alleen in één richting. Het installeert, actualiseert of downloadt niets en
  rapporteert geen telemetrie over installaties/downloads.

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- Vereist `clawhub login`.
- Voert ClawHub ClawScan uit via `POST /api/v1/skills/-/scan` en pollt vervolgens totdat de scan een eindstatus heeft.
- Scans zijn asynchroon en kunnen enige tijd duren. In de wachtrij toont de spinner in de terminal de huidige geprioriteerde scanpositie en hoeveel scans ervoor staan.
- Voor scans van gepubliceerde versies is eigendom of beheerderstoegang voor de uitgever vereist. Moderators/beheerders kunnen dezelfde backend gebruiken via `clawhub-admin`.
- `--update` is alleen geldig met `--slug`; hiermee worden succesvolle scanresultaten voor gepubliceerde versies teruggeschreven naar de geselecteerde versie.
- `--output <file.zip>` downloadt het volledige rapportarchief met `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json` en `README.md`.
- `--json` drukt het volledige pollingantwoord af voor automatisering.
- Scans van lokale paden worden niet langer ondersteund. Upload een nieuwe versie en gebruik vervolgens `scan download` om de opgeslagen scanresultaten voor die ingediende versie op te halen.

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- Vereist `clawhub login`.
- Downloadt het opgeslagen ZIP-bestand met het scanrapport voor een ingediende skill- of pluginversie, inclusief versies die door beveiligingscontroles van ClawHub zijn geblokkeerd of verborgen.
- Downloads van skills gebruiken de skill-slug en zijn standaard ingesteld op `--kind skill`.
- Downloads van plugins gebruiken de pakketnaam en vereisen `--kind plugin`.
- `--version` is vereist, zodat auteurs precies de ingediende versie inspecteren die ClawHub heeft geblokkeerd.
- `--output <file.zip>` kiest het doelpad.

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub levert een officiële herbruikbare workflow op
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
voor skill-repository's en catalogusrepository's.

Gebruikelijke catalogusconfiguratie:

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Opmerkingen:

- `root` is voor catalogusrepository's standaard ingesteld op `skills`.
- Geef `skill_path: skills/review-helper` door om één skillmap te verwerken.
- `owner` komt overeen met de CLI-vlag `--owner`; laat deze weg om als de geauthenticeerde gebruiker te publiceren.
- Voor het publiceren van V1-skills wordt `clawhub_token` gebruikt; vertrouwd publiceren met GitHub OIDC is voorlopig alleen beschikbaar voor pakketten.

### `delete <skill>`

- Zonder `--version` wordt een skill voorlopig verwijderd (door de eigenaar, een moderator of een beheerder).
- Roept `DELETE /api/v1/skills/{slug}` aan.
- Door de eigenaar geïnitieerde voorlopige verwijderingen reserveren de slug 30 dagen; de opdracht toont de vervaltijd.
- `--version <version>` trekt één eigen versie die niet de nieuwste is in via een fail-closed,
  versiespecifieke route. Het versienummer blijft gereserveerd en kan niet opnieuw worden gepubliceerd met
  een andere inhoud. Publiceer een vervanging voordat je de huidige nieuwste versie verwijdert. Platform-
  medewerkers omzeilen het eigenaarschap niet voor deze versiegebonden flow.
- `--reason <text>` legt een moderatienotitie vast bij een voorlopige verwijdering van de volledige skill en in het auditlogboek.
- `--note <text>` is een alias voor `--reason`.
- `--yes` slaat de bevestiging over.

### `undelete <skill>`

- Herstelt een verborgen skill (door de eigenaar, een moderator of een beheerder).
- Roept `POST /api/v1/skills/{slug}/undelete` aan.
- `--version <version>` herstelt uitsluitend exact het behouden artefact dat eerder door dezelfde
  eigenaar-actor is ingetrokken. De herstelde versie wordt hierdoor niet de nieuwste en verwijderde tags worden niet opnieuw aangemaakt.
- Versieherstel roept `POST /api/v1/skills/{slug}/versions/{version}/restore` aan.
- `--reason <text>` legt een moderatienotitie vast bij de skill en in het auditlogboek.
- `--note <text>` is een alias voor `--reason`.
- `--yes` slaat de bevestiging over.

### `hide <skill>`

- Verbergt een skill (door de eigenaar, een moderator of een beheerder).
- Alias voor `delete`.

### `unhide <skill>`

- Maakt een skill weer zichtbaar (door de eigenaar, een moderator of een beheerder).
- Alias voor `undelete`.

### `skill rename <skill> <new-name>`

- Wijzigt de naam van een eigen skill en behoudt de vorige slug als omleidingsalias.
- Roept `POST /api/v1/skills/{slug}/rename` aan.
- `--yes` slaat de bevestiging over.

### `skill merge <source> <target>`

- Voegt één eigen skill samen met een andere eigen skill.
- De bronslug wordt niet meer openbaar weergegeven en wordt een omleidingsalias naar het doel.
- Roept `POST /api/v1/skills/{sourceSlug}/merge` aan.
- `--yes` slaat de bevestiging over.

### `transfer`

- Workflow voor eigendomsoverdracht.
- Overdrachten naar gebruikershandles maken een openstaand verzoek aan dat de ontvanger accepteert.
- Overdrachten naar organisatie-/uitgevershandles worden alleen onmiddellijk toegepast wanneer de actor
  beheerderstoegang heeft tot zowel de huidige eigenaar als de doeluitgever.
- Subopdrachten:
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- Eindpunten:
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- Bladert door of doorzoekt de uniforme pakketcatalogus via `GET /api/v1/packages` en `GET /api/v1/packages/search`.
- Gebruik dit voor plugins en andere vermeldingen uit pakketfamilies; `search` op het hoogste niveau blijft de zoekfunctie voor skills.
- Vlaggen:
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`, `--os <os>`, `--arch <arch>`, `--libc <libc>`
  - `--requires-browser`, `--requires-desktop`, `--requires-native-deps`
  - `--requires-external-service`, `--external-service <name>`
  - `--binary <name>`, `--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>` (1-100, standaard: 25)
  - `--json`

Voorbeelden:

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- Haalt pakketmetadata op zonder te installeren.
- Gebruik dit om pluginmetadata, compatibiliteit, verificatie, broncode en versies/bestanden te inspecteren.
- `--version <version>`: inspecteert een specifieke versie (standaard: nieuwste).
- `--tag <tag>`: inspecteert een getagde versie (bijv. `latest`).
- `--versions`: geeft de versiegeschiedenis weer (eerste pagina).
- `--limit <n>`: maximumaantal weer te geven versies (1-100).
- `--files`: geeft bestanden voor de geselecteerde versie weer.
- `--file <path>`: haalt een begrensd UTF-8-tekstvoorbeeld op (limiet van 200KB).
- `--json`: machineleesbare uitvoer.

### `package download <name>`

- Bepaalt een pakketversie via
  `GET /api/v1/packages/{name}/versions/{version}/artifact`.
- Downloadt het artefact vanuit `downloadUrl` van de resolver.
- Verifieert ClawHub SHA-256 voor alle artefacten.
- Voor ClawPack npm-pack-artefacten worden ook de integriteit van npm `sha512`,
  de npm-shasum en de naam/versie in `package.json` van de tarball geverifieerd.
- Oudere ZIP-versies worden via de oudere ZIP-route gedownload.
- Vlaggen:
  - `--version <version>`: downloadt een specifieke versie.
  - `--tag <tag>`: downloadt een getagde versie (standaard: `latest`).
  - `-o, --output <path>`: uitvoerbestand of -map.
  - `--force`: overschrijft een bestaand uitvoerbestand.
  - `--json`: machineleesbare uitvoer.

Voorbeelden:

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- Berekent de ClawHub SHA-256, de integriteit van npm `sha512` en de npm-shasum voor een lokaal
  artefact.
- Met `--package` worden de verwachte metadata uit ClawHub bepaald en wordt het
  lokale bestand vergeleken met de metadata van het gepubliceerde artefact.
- Met rechtstreekse digest-vlaggen wordt geverifieerd zonder netwerkzoekopdracht.
- Vlaggen:
  - `--package <name>`: pakketnaam om de verwachte artefactmetadata te bepalen.
  - `--version <version>` of `--tag <tag>`: verwachte pakketversie.
  - `--sha256 <hex>`: verwachte ClawHub SHA-256.
  - `--npm-integrity <sri>`: verwachte npm-integriteit.
  - `--npm-shasum <sha1>`: verwachte npm-shasum.
  - `--json`: machineleesbare uitvoer.

Voorbeelden:

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- Voert de meegeleverde Plugin Inspector van de ClawHub-CLI uit op een lokale map met een
  pluginpakket.
- Gebruikt standaard offline/statische validatie, zonder een lokale OpenClaw-checkout te zoeken
  of te importeren.
- Harde compatibiliteitsfouten resulteren in een afsluiting met een niet-nulwaarde. Bevindingen die alleen waarschuwingen bevatten, worden weergegeven maar
  resulteren in een afsluiting met nul.
- Vlaggen:
  - `--out <dir>`: schrijft rapporten van Plugin Inspector naar deze map.
  - `--openclaw <path>`: inspecteert aan de hand van een expliciete lokale OpenClaw-checkout.
  - `--runtime`: schakelt runtimevastlegging in; importeert plugincode.
  - `--allow-execute`: staat runtimevastlegging toe in een geïsoleerde werkruimte.
  - `--no-mock-sdk`: schakelt de gesimuleerde OpenClaw-SDK uit tijdens runtimevastlegging.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package validate ./example-plugin
```

Als de validatie een bevinding over een pakket, manifest, SDK-import of artefact meldt, raadpleeg dan
[Oplossingen voor pluginvalidatie](/nl/clawhub/plugin-validation-fixes) en voer de opdracht opnieuw uit.

### `package delete <name>`

- Zonder `--version` worden een pakket en alle releases voorlopig verwijderd.
- `--version <version>` trekt één eigen release die niet de nieuwste is in via een fail-closed,
  versiespecifieke route. Het versienummer blijft gereserveerd en kan niet opnieuw worden gepubliceerd met
  een andere inhoud. Publiceer een vervanging voordat je de huidige nieuwste versie verwijdert. Deze
  versiegebonden flow vereist de pakketeigenaar of een beheerder van de organisatie-uitgever; platformmedewerkers
  omzeilen het pakketeigenaarschap niet.
- Voorlopige verwijdering van het volledige pakket vereist de pakketeigenaar, een eigenaar/beheerder van de organisatie-uitgever, een platform-
  moderator of een platformbeheerder.
- Vlaggen:
  - `--version <version>`: trekt één versie in die niet de nieuwste is.
  - `--yes`: slaat de bevestiging over.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- Herstelt een voorlopig verwijderd pakket en de releases.
- Vereist de pakketeigenaar, een eigenaar/beheerder van de organisatie-uitgever, een platformmoderator
  of een platformbeheerder.
- Roept `POST /api/v1/packages/{name}/undelete` aan.
- `--version <version>` herstelt uitsluitend exact de behouden release die eerder door dezelfde
  eigenaar-actor is ingetrokken. De release wordt hierdoor niet de nieuwste en verwijderde pakkettags/dist-tags worden niet opnieuw aangemaakt.
- Versieherstel roept `POST /api/v1/packages/{name}/versions/{version}/restore` aan.
- Vlaggen:
  - `--version <version>`: herstelt één door de eigenaar ingetrokken release.
  - `--yes`: slaat de bevestiging over.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- Draagt een pakket over aan een andere uitgever.
- Vereist beheerderstoegang tot zowel de huidige pakketeigenaar als de
  doeluitgever, tenzij dit wordt uitgevoerd door een platformbeheerder.
- Pakketnamen met een bereik moeten worden overgedragen aan de eigenaar van het overeenkomende bereik.
- Roept `POST /api/v1/packages/{name}/transfer` aan.
- Vlaggen:
  - `--to <owner>`: handle van de doeluitgever.
  - `--reason <text>`: optionele auditreden.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- Geverifieerde opdracht om een pakket bij moderators te melden.
- Roept `POST /api/v1/packages/{name}/report` aan.
- Meldingen gelden op pakketniveau, kunnen optioneel aan een versie worden gekoppeld en worden zichtbaar
  voor moderators ter beoordeling.
- Meldingen verbergen pakketten niet automatisch en blokkeren op zichzelf geen downloads.
- Vlaggen:
  - `--version <version>`: optionele pakketversie om aan de melding te koppelen.
  - `--reason <text>`: verplichte reden voor de melding.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "verdachte native payload"
```

### `package moderation-status`

- Opdracht voor eigenaren om de zichtbaarheid van pakketmoderatie te controleren.
- Roept `GET /api/v1/packages/{name}/moderation` aan.
- Toont de huidige scanstatus van het pakket, het aantal openstaande meldingen, de handmatige
  moderatiestatus van de nieuwste release, de downloadblokkeringsstatus en moderatieredenen.
- Vlaggen:
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- Controleert of een pakket gereed is voor toekomstig gebruik door OpenClaw.
- Roept `GET /api/v1/packages/{name}/readiness` aan.
- Meldt blokkades voor officiële status, beschikbaarheid van ClawPack, artefactdigest,
  bronherkomst, compatibiliteit met OpenClaw, hostdoelen, omgevingsmetadata
  en scanstatus.
- Vlaggen:
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- Toont een op operators gerichte migratiestatus voor een pakket dat mogelijk een
  gebundelde OpenClaw-plugin vervangt.
- Roept hetzelfde berekende gereedheidseindpunt aan als `package readiness`, maar toont
  een op migratie gerichte status, de nieuwste versie, de status van het officiële pakket, controles en
  blokkades.
- Vlaggen:
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- Maakt een organisatie-uitgever aan die eigendom is van de geverifieerde gebruiker.
- De handle wordt genormaliseerd naar kleine letters en kan met of zonder `@` worden doorgegeven.
- Nieuw aangemaakte organisatie-uitgevers zijn standaard niet vertrouwd/officieel.
- Mislukt als de handle al wordt gebruikt door een bestaande uitgever, gebruiker of gereserveerde route.

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- Publiceert een codeplugin of bundelplugin via `POST /api/v1/packages`.
- `<source>` accepteert:
  - Lokaal mappad: `./my-plugin`
  - Lokale npm-pack-tarball van ClawPack: `./my-plugin-1.2.3.tgz`
  - GitHub-repository: `owner/repo` of `owner/repo@ref`
  - GitHub-URL: `https://github.com/owner/repo`
- Metadata wordt automatisch gedetecteerd uit `package.json`, `openclaw.plugin.json` en
  echte OpenClaw-bundelmarkeringen zoals `.codex-plugin/plugin.json`,
  `.claude-plugin/plugin.json` en `.cursor-plugin/plugin.json`.
- `.tgz`-bronnen worden behandeld als ClawPack. De CLI uploadt de exacte npm-pack-
  bytes en gebruikt de uitgepakte inhoud van `package/` alleen voor validatie en
  het vooraf invullen van metadata.
- Mappen met codeplugins worden vóór het uploaden verpakt in een ClawPack-npm-tarball, zodat
  OpenClaw-installaties het exacte artefact kunnen verifiëren. Mappen met bundelplugins
  gebruiken nog steeds het publicatiepad voor uitgepakte bestanden.
- Voor GitHub-bronnen wordt bronvermelding automatisch ingevuld op basis van de repository, de bepaalde commit, de ref en het subpad.
- Voor lokale mappen wordt bronvermelding automatisch gedetecteerd vanuit lokale git wanneer de externe origin naar GitHub verwijst.
- Externe codeplugins moeten `openclaw.compat.pluginApi` en
  `openclaw.build.openclawVersion` expliciet declareren.
  `package.json.version` op het hoogste niveau wordt niet gebruikt als terugvaloptie voor publicatievalidatie.
- `--dry-run` toont een voorbeeld van de bepaalde publicatiepayload zonder deze te uploaden.
- `--json` produceert machineleesbare uitvoer voor CI.
- `--owner <handle>` publiceert onder de handle van een gebruiker of organisatie-uitgever wanneer de actor uitgeverstoegang heeft.
- Pakketnamen met een bereik moeten overeenkomen met de geselecteerde eigenaar. Zie `docs/publishing.md`.
- Bestaande vlaggen (`--family`, `--name`, `--version`, `--source-repo`, `--source-commit`, `--source-ref`, `--source-path`) blijven werken als overschrijvingen.
- Privé-GitHub-repository's vereisen `GITHUB_TOKEN`.

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### Aanbevolen lokale werkwijze

Gebruik eerst `--dry-run`, zodat je de bepaalde pakketmetadata en
bronvermelding kunt bevestigen voordat je een live-release maakt:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### Werkwijze voor lokale mappen

Bij codeplugins bouwt en uploadt publicatie vanuit een map een ClawPack-artefact vanuit
de pakketmap:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### Minimale `package.json` voor `--family code-plugin`

Externe codeplugins hebben een kleine hoeveelheid OpenClaw-metadata nodig in
`package.json`. Dit minimale manifest volstaat voor een geslaagde publicatie:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

Verplichte velden:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

Opmerkingen:

- `package.json.version` is de releaseversie van je pakket, maar wordt niet gebruikt als
  terugvaloptie voor compatibiliteits-/buildvalidatie van OpenClaw.
- `openclaw.hostTargets` en `openclaw.environment` zijn optionele metadata.
  ClawHub kan deze tonen als ze aanwezig zijn, maar ze zijn niet vereist voor publicatie.
- `openclaw.compat.minGatewayVersion` en
  `openclaw.build.pluginSdkVersion` zijn optionele aanvullingen als je
  gedetailleerdere compatibiliteitsmetadata wilt publiceren.
- Als je een oudere release van de `clawhub`-CLI gebruikt, voer dan vóór publicatie een upgrade uit, zodat
  de lokale preflight-controles vóór het uploaden worden uitgevoerd.
- Als de validatie een herstelcode meldt, zie dan
  [Oplossingen voor pluginvalidatie](/nl/clawhub/plugin-validation-fixes).

#### GitHub Actions

ClawHub levert ook een officiële herbruikbare workflow op
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
voor pluginrepository's.

Typische configuratie van de aanroeper:

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Opmerkingen:

- De herbruikbare workflow stelt `source` standaard in op de repository van de aanroeper.
- Geef voor monorepository's `source_path` door, zodat de workflow de
  pakketmap van de plugin publiceert, bijvoorbeeld `source_path: extensions/codex`.
- Zet de herbruikbare workflow vast op een stabiele tag of volledige commit-SHA. Voer releasepublicaties niet uit vanaf `@main`.
- `pull_request` moet `dry_run: true` gebruiken, zodat CI geen vervuiling veroorzaakt.
- Echte publicaties moeten worden beperkt tot vertrouwde gebeurtenissen, zoals `workflow_dispatch` of tag-pushes.
- Vertrouwd publiceren zonder geheim werkt alleen op `workflow_dispatch`; tag-pushes vereisen nog steeds `clawhub_token`.
- Houd `clawhub_token` beschikbaar voor de eerste publicatie, niet-vertrouwde pakketten of noodpublicaties.
- De workflow uploadt het JSON-resultaat als artefact en stelt het beschikbaar als workflowuitvoer.

### `package trusted-publisher get <name>`

- Toont de configuratie van de vertrouwde GitHub Actions-uitgever voor een pakket.
- Gebruik dit na het instellen van de configuratie om de repository, de naam van het workflowbestand
  en de optionele omgevingsvergrendeling te bevestigen.
- Vlaggen:
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- Koppelt de configuratie van een vertrouwde GitHub Actions-uitgever aan een bestaand
  pakket of vervangt deze.
- Het pakket moet eerst worden aangemaakt via normale handmatige of met een token geverifieerde
  `clawhub package publish`.
- Nadat de configuratie is ingesteld, kunnen toekomstige ondersteunde GitHub Actions-publicaties
  OIDC/vertrouwde publicatie gebruiken zonder een langlevend ClawHub-token.
- `--repository <repo>` moet `owner/repo` zijn.
- `--workflow-filename <file>` moet overeenkomen met de naam van het workflowbestand in
  `.github/workflows/`.
- `--environment <name>` is optioneel. Als dit is geconfigureerd, moet de GitHub Actions-
  omgeving in de OIDC-claim exact overeenkomen.
- ClawHub verifieert de geconfigureerde GitHub-repository wanneer deze opdracht wordt uitgevoerd.
  Openbare repository's kunnen via openbare GitHub-metadata worden geverifieerd. Voor privé-
  repository's moet ClawHub GitHub-toegang tot die repository hebben,
  bijvoorbeeld via een toekomstige installatie van de ClawHub GitHub App of een andere geautoriseerde
  GitHub-integratie.
- Vlaggen:
  - `--repository <repo>`: GitHub-repository, bijvoorbeeld `openclaw/example-plugin`.
  - `--workflow-filename <file>`: naam van het workflowbestand, bijvoorbeeld `package-publish.yml`.
  - `--environment <name>`: optionele GitHub Actions-omgeving die exact moet overeenkomen.
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- Verwijdert de configuratie van de vertrouwde uitgever uit een pakket.
- Gebruik dit als terugdraaioptie als de workflow, repository of omgevingsvergrendeling moet
  worden uitgeschakeld of opnieuw aangemaakt.
- Toekomstige echte publicaties moeten normale geverifieerde publicatie gebruiken totdat de configuratie
  opnieuw is ingesteld.
- Vlaggen:
  - `--json`: machineleesbare uitvoer.

Voorbeeld:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### Installatietelemetrie

- Wordt na `clawhub install <slug>` verzonden wanneer je bent ingelogd, tenzij
  `CLAWHUB_DISABLE_TELEMETRY=1` is ingesteld.
- Rapportage gebeurt naar beste vermogen. Installatieopdrachten mislukken niet als telemetrie
  niet beschikbaar is.
- Details: `docs/telemetry.md`.
