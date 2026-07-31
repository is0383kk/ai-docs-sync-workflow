---
read_when:
    - Je wilt Gateway-plugins of compatibele bundels installeren of beheren
    - Je wilt een eenvoudige toolplugin opzetten of valideren
    - Je wilt fouten bij het laden van plugins opsporen
sidebarTitle: Plugins
summary: CLI-referentie voor `openclaw plugins` (initialiseren, bouwen, valideren, weergeven, installeren, marketplace, verwijderen, inschakelen/uitschakelen, doctor)
title: Plugins
x-i18n:
    generated_at: "2026-07-27T06:09:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

Beheer Gateway-plugins, hookpakketten en compatibele bundels.

<CardGroup cols={2}>
  <Card title="Pluginsysteem" href="/nl/tools/plugin">
    Handleiding voor eindgebruikers voor het installeren, inschakelen en oplossen van problemen met plugins.
  </Card>
  <Card title="Plugins beheren" href="/nl/plugins/manage-plugins">
    Korte voorbeelden voor installeren, weergeven, bijwerken, verwijderen en publiceren.
  </Card>
  <Card title="Pluginbundels" href="/nl/plugins/bundles">
    Compatibiliteitsmodel voor bundels.
  </Card>
  <Card title="Pluginmanifest" href="/nl/plugins/manifest">
    Manifestvelden en configuratieschema.
  </Card>
  <Card title="Beveiliging" href="/nl/gateway/security">
    Beveiligingsversterking voor plugininstallaties.
  </Card>
</CardGroup>

## Opdrachten

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # alias voor inspect
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

Voer voor onderzoek naar trage installaties, inspecties, verwijderingen of registervernieuwingen de
opdracht uit met `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1`. De tracering schrijft fasetijden
naar stderr en houdt JSON-uitvoer parseerbaar. Zie [Foutopsporing](/nl/help/debugging#plugin-lifecycle-trace).

<Note>
In Nix-modus (`OPENCLAW_NIX_MODE=1`) is `openclaw.json` onveranderlijk. `install`, `update`, `uninstall`, `enable` en `disable` weigeren allemaal te worden uitgevoerd. Bewerk in plaats daarvan de Nix-bron voor deze installatie (`programs.openclaw.config` of `instances.<name>.config` voor nix-openclaw) en bouw vervolgens opnieuw. Zie de agentgerichte [Snelstart](https://github.com/openclaw/nix-openclaw#quick-start).
</Note>

<Note>
Gebundelde plugins worden met OpenClaw meegeleverd. Sommige zijn standaard ingeschakeld (bijvoorbeeld gebundelde modelproviders, gebundelde spraakproviders en de gebundelde browserplugin); andere vereisen `plugins enable`.

Native OpenClaw-plugins leveren `openclaw.plugin.json` met een inline JSON-schema (`configSchema`, zelfs als dit leeg is). Compatibele bundels gebruiken in plaats daarvan hun eigen bundelmanifesten.

`plugins list` toont `Format: openclaw` of `Format: bundle`. Uitgebreide lijst-/info-uitvoer toont ook het bundelsubtype (`codex`, `claude` of `cursor`) plus de gedetecteerde bundelmogelijkheden.
</Note>

## Maken

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` maakt standaard een minimale TypeScript-toolplugin. Het eerste
argument is de plugin-id; `--name` stelt de weergavenaam in. OpenClaw gebruikt de
id voor de standaarduitvoermap en pakketnaamgeving. Toolsteigers gebruiken
`defineToolPlugin` en genereren `package.json`-scripts `plugin:build` en
`plugin:validate`, die eerst bouwen en daarna `openclaw plugins build`/`validate` aanroepen.

`plugins build` importeert het gebouwde toegangspunt, leest de statische toolmetadata, schrijft
`openclaw.plugin.json` en houdt `package.json` van `openclaw.extensions` daarmee in overeenstemming.
`plugins validate` controleert of het gegenereerde manifest, de pakketmetadata en
de huidige export van het toegangspunt nog steeds overeenkomen. Zie [Toolplugins](/nl/plugins/tool-plugins) voor
de volledige workflow voor auteurs.

De steiger schrijft TypeScript-broncode, maar genereert metadata vanuit het gebouwde
`./dist/index.js`-toegangspunt, zodat de workflow ook werkt met de gepubliceerde CLI. Gebruik
`--entry <path>` wanneer het toegangspunt niet het standaardtoegangspunt van het pakket is. Gebruik
`plugins build --check` in CI om te mislukken wanneer gegenereerde metadata verouderd is, zonder
bestanden te herschrijven.

### Providersteiger

```bash
openclaw plugins init acme-models --name "Acme Models" --type provider
cd acme-models
npm install
npm run build
npm test
npm run validate
```

Providersteigers maken een generieke, met OpenAI compatibele modelproviderplugin
met API-sleutelauthenticatie, een `npm run validate`-script dat
`clawhub package validate` uitvoert, ClawHub-pakketmetadata en een handmatig
gestarte GitHub Actions-workflow voor toekomstige vertrouwde publicatie via GitHub
OIDC. Providersteigers genereren geen Skills en gebruiken
`openclaw plugins build`/`validate` niet; die opdrachten zijn bedoeld voor het pad met gegenereerde metadata
van de toolsteiger.

Vervang vóór publicatie de tijdelijke API-basis-URL, modelcatalogus, documentatieroute,
referentietekst en README-tekst door echte providergegevens. Gebruik de
gegenereerde README voor de eerste publicatie op ClawHub en het instellen van een vertrouwde uitgever.

## Installeren

```bash
openclaw plugins search "calendar"                      # zoeken naar ClawHub-plugins
openclaw plugins install @openclaw/<package>            # vertrouwde officiële catalogus
openclaw plugins install <package>                       # willekeurig npm-pakket
openclaw plugins install clawhub:<package>                # alleen ClawHub
openclaw plugins install npm:<package>                    # alleen npm
openclaw plugins install npm-pack:<path.tgz>               # lokaal npm-pack-tarbestand
openclaw plugins install git:github.com/<owner>/<repo>     # git-repository
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <path>                            # lokaal pad of archief
openclaw plugins install -l <path>                         # koppelen in plaats van kopiëren
openclaw plugins install <plugin>@<marketplace>             # verkorte marketplace-notatie
openclaw plugins install <plugin> --marketplace <name>      # marketplace (expliciet)
openclaw plugins install <package> --force                  # bron bevestigen / bestaande installatie overschrijven
openclaw plugins install <package> --pin                    # opgeloste npm-versie vastzetten
openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
openclaw plugins install <package> --dangerously-force-unsafe-install
```

Beheerders die installaties tijdens het instellen testen, kunnen automatische bronnen voor
plugininstallaties overschrijven met beveiligde omgevingsvariabelen. Zie
[Overschrijvingen voor plugininstallaties](/nl/plugins/install-overrides).

<Warning>
Losse pakketnamen worden tijdens de overgang bij de lancering standaard vanuit npm geïnstalleerd, tenzij ze overeenkomen met de id van een gebundelde of officiële plugin. In dat geval gebruikt OpenClaw die lokale/officiële kopie in plaats van het npm-register te benaderen. Gebruik `npm:<package>` wanneer je bewust een extern npm-pakket wilt gebruiken. Gebruik `clawhub:<package>` voor ClawHub. Behandel plugininstallaties als het uitvoeren van code; geef de voorkeur aan vastgezette versies.
</Warning>

<Warning>
ClawHub-pakketten en de gebundelde/officiële catalogus van OpenClaw zijn vertrouwde
installatiebronnen. Een nieuwe willekeurige npm-, `npm-pack:`-, git-, lokale pad-/archief- of
marketplace-bron geeft een waarschuwing en vraagt om bevestiging voordat wordt doorgegaan. Niet-interactieve willekeurige
installaties moeten `--force` doorgeven nadat je de bron hebt gecontroleerd en vertrouwt. Dezelfde
vlag overschrijft indien nodig een bestaand installatiedoel. Normale updates van een
reeds bijgehouden installatie vereisen dit niet. Deze bevestiging staat los van
`--acknowledge-clawhub-risk`, dat alleen van toepassing is op vertrouwenswaarschuwingen voor risicovolle
ClawHub-releases. `--force` omzeilt `security.installPolicy` of de overige
installatiebeveiligingscontroles niet.
</Warning>

`plugins search` bevraagt ClawHub naar installeerbare `code-plugin`- en
`bundle-plugin`-pakketten (geen Skills; gebruik daarvoor `openclaw skills search`).
De standaardwaarde van `--limit` is 20, met een maximum van 100. Het leest alleen de externe catalogus: geen
inspectie van lokale status, configuratiewijziging, pakketinstallatie of laden van de pluginruntime.
Resultaten bevatten de ClawHub-pakketnaam, familie, het kanaal, de versie,
samenvatting en een installatiehint zoals `openclaw plugins install clawhub:<package>`.

<Note>
ClawHub is voor de meeste plugins het primaire distributie- en ontdekkingsoppervlak. Npm
blijft een ondersteund terugval- en direct installatiepad. Pluginpakketten van OpenClaw
`@openclaw/*` worden opnieuw op npm gepubliceerd; zie de huidige lijst
op [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) of de
[plugininventaris](/nl/plugins/plugin-inventory). Stabiele installaties gebruiken `latest`.
Installaties en updates via het bètakanaal geven waar beschikbaar de voorkeur aan de npm-dist-tag `beta`
en vallen anders terug op `latest`. Op het extended-stable-kanaal worden officiële npm-plugins
met een losse/standaard- of `latest`-intentie omgezet naar exact de geïnstalleerde kernversie.
Exacte vastzettingen en expliciete niet-`latest`-tags, pakketten van derden en
niet-npm-bronnen worden niet herschreven.
</Note>

<AccordionGroup>
  <Accordion title="Configuratie-includes en herstel van ongeldige configuratie">
    Als je `plugins`-sectie wordt geleverd via een `$include` met één bestand, schrijft `plugins install/update/enable/disable/uninstall` door naar dat opgenomen bestand en laat het `openclaw.json` ongemoeid. Root-includes, include-arrays en includes met naastliggende overschrijvingen stoppen veilig in plaats van ze samen te voegen. Zie [Configuratie-includes](/nl/gateway/configuration) voor de ondersteunde vormen.

    Als de configuratie vóór de installatie ongeldig is, stopt `plugins install` normaal gesproken veilig en wordt aangegeven dat je eerst `openclaw doctor --fix` moet uitvoeren. Tijdens het starten en hot-reloaden van de Gateway stopt ongeldige pluginconfiguratie veilig, net als elke andere ongeldige configuratie; `openclaw doctor --fix` kan de ongeldige pluginvermelding in quarantaine plaatsen. De enige uitzondering voor vooraf bestaande configuratie is een beperkt herstelpad voor gebundelde plugins die expliciet kiezen voor `openclaw.install.allowInvalidConfigRecovery`.

    Wanneer de bestaande hostconfiguratie geldig is, maar de eigen configuratie van de nieuw geïnstalleerde plugin ontbreekt, registreert OpenClaw de installatie als uitgeschakeld in plaats van een ongeldige ingeschakelde vermelding te schrijven. Configureer `plugins.entries.<id>.config` en voer daarna `openclaw plugins enable <id>` uit. Als er al een pluginconfiguratievermelding bestaat maar deze ongeldig is, mislukt de installatie zonder deze te herschrijven.

  </Accordion>
  <Accordion title="--force-bevestiging en opnieuw installeren versus bijwerken">
    `--force` bevestigt een niet-ClawHub-bron zonder hierom te vragen. Het omzeilt `security.installPolicy` of de overige installatiebeveiligingscontroles niet. Wanneer de plugin of het hookpakket al is geïnstalleerd, wordt ook het bestaande doel hergebruikt en ter plaatse overschreven. Gebruik dit nadat je een willekeurige npm-, lokale, archief-, git- of marketplace-bron hebt gecontroleerd, of wanneer je bewust dezelfde id opnieuw installeert. Geef voor routinematige upgrades van een reeds bijgehouden npm-plugin de voorkeur aan `openclaw plugins update <id-or-npm-spec>`.

    Als je `plugins install` uitvoert voor een plugin-id die al is geïnstalleerd, stopt OpenClaw en verwijst het je naar `plugins update <id-or-npm-spec>` voor een normale upgrade, of naar `plugins install <package> --force` wanneer je de huidige installatie werkelijk vanuit een andere bron wilt overschrijven. Willekeurige bronnen tonen nog steeds de interactieve herkomstwaarschuwing; niet-interactieve installaties moeten na controle `--force` doorgeven. Vertrouwde ClawHub- en OpenClaw-catalogusbronnen hebben dit niet nodig. Met `--link` bevestigt `--force` de bron, maar wijzigt het de installatiemodus met gekoppeld pad niet.

  </Accordion>
  <Accordion title="Bereik van --pin">
    `--pin` is alleen van toepassing op npm-installaties en legt de exact opgeloste `<name>@<version>` vast. Dit wordt niet ondersteund bij `git:`-installaties (zet de ref in plaats daarvan vast in de specificatie, bijvoorbeeld `git:github.com/acme/plugin@v1.2.3`) of bij `--marketplace` (marketplace-installaties slaan marketplace-bronmetadata op in plaats van een npm-specificatie).
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` is verouderd en doet nu niets meer. OpenClaw voert bij Plugin-installaties niet langer ingebouwde blokkering van gevaarlijke code tijdens de installatie uit.

    Gebruik het door de operator beheerde `security.installPolicy`-oppervlak wanneer hostspecifiek installatiebeleid vereist is. Plugin-`before_install`-hooks zijn levenscyclushooks voor de Plugin-runtime, niet de primaire beleidsgrens voor CLI-installaties.

    Als een Plugin die je op ClawHub hebt gepubliceerd, wordt verborgen of geblokkeerd door een registerscan, gebruik dan de stappen voor uitgevers in [Publiceren op ClawHub](/nl/clawhub/publishing). `--dangerously-force-unsafe-install` vraagt ClawHub niet om de Plugin opnieuw te scannen of een geblokkeerde release openbaar te maken.

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    Installaties vanuit de ClawHub-community controleren vóór het downloaden het vertrouwensrecord van de geselecteerde release. Als ClawHub downloaden voor de release uitschakelt, schadelijke scanbevindingen meldt of de release in een blokkerende moderatiestatus plaatst (in quarantaine geplaatst, ingetrokken), weigert OpenClaw deze zonder uitzondering, ongeacht deze vlag. Bij niet-blokkerende risicovolle scanstatussen of moderatiestatussen toont OpenClaw de vertrouwensdetails en vraagt het vóór het doorgaan om bevestiging.

    Gebruik `--acknowledge-clawhub-risk` alleen nadat je de ClawHub-waarschuwing hebt beoordeeld en hebt besloten zonder interactieve prompt door te gaan. Wachtende of verouderde (nog niet schone) scanresultaten geven een waarschuwing, maar vereisen geen bevestiging. Officiële ClawHub-pakketten en meegeleverde OpenClaw Plugin-bronnen slaan deze controle van het releasevertrouwen volledig over.

  </Accordion>
  <Accordion title="Hookpakketten en npm-specificaties">
    `plugins install` is ook het installatieoppervlak voor hookpakketten die `openclaw.hooks` beschikbaar stellen in `package.json`. Gebruik `openclaw hooks` voor gefilterde zichtbaarheid van hooks en inschakeling per hook, niet voor pakketinstallatie.

    Npm-specificaties zijn **alleen voor registers** (pakketnaam plus optioneel een **exacte versie** of **dist-tag**). Git-/URL-/bestandsspecificaties en semver-bereiken worden geweigerd. Afhankelijkheden worden voor de veiligheid geïnstalleerd in één beheerd npm-project per Plugin met `--ignore-scripts`, zelfs wanneer je shell algemene npm-installatie-instellingen heeft. Beheerde npm-projecten voor Plugins nemen de npm-`overrides` op pakketniveau van OpenClaw over, zodat beveiligingsvastzettingen van de host ook van toepassing zijn op omhooggehesen Plugin-afhankelijkheden.

    Gebruik `npm:<package>` om npm-resolutie expliciet te maken. Kale pakketspecificaties worden tijdens de overgang bij lancering ook rechtstreeks vanuit npm geïnstalleerd, tenzij ze overeenkomen met een officiële Plugin-id.

    Onbewerkte `@openclaw/*`-specificaties die overeenkomen met meegeleverde Plugins, worden vóór de npm-terugval omgezet naar de meegeleverde kopie die bij de image hoort. `openclaw plugins install @openclaw/discord@2026.5.20 --pin` gebruikt bijvoorbeeld de meegeleverde Discord-Plugin uit de huidige OpenClaw-build in plaats van een beheerde npm-overschrijving te maken. Gebruik `openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin` om het externe npm-pakket af te dwingen.

    Kale specificaties en `@latest` blijven op het stabiele kanaal. OpenClaw-correctieversies met een datumstempel, zoals `2026.5.3-1`, gelden voor deze controle als stabiel. Als npm een van beide vormen omzet naar een prerelease, stopt OpenClaw en vraagt het je expliciet in te stemmen met een prerelease-tag (`@beta`/`@rc`) of een exacte prereleaseversie (`@1.2.3-beta.4`).

    Bij npm-installaties zonder exacte versie (`npm:<package>` of `npm:<package>@latest`) controleert OpenClaw vóór de installatie de metadata van het opgeloste pakket. Als het nieuwste stabiele pakket een nieuwere OpenClaw Plugin-API of minimale hostversie vereist, inspecteert OpenClaw oudere stabiele versies en installeert het in plaats daarvan de nieuwste compatibele release. Exacte versies en expliciete dist-tags blijven strikt: een incompatibele selectie mislukt en vraagt je OpenClaw te upgraden of een compatibele versie te kiezen.

    Als een kale installatiespecificatie overeenkomt met een officiële Plugin-id (bijvoorbeeld `diffs`), installeert OpenClaw rechtstreeks het catalogusitem. Gebruik een expliciete specificatie met bereik om een npm-pakket met dezelfde naam te installeren (bijvoorbeeld `@scope/diffs`).

  </Accordion>
  <Accordion title="Git-repository's">
    Gebruik `git:<repo>` om rechtstreeks vanuit een git-repository te installeren. Ondersteunde vormen: `git:github.com/owner/repo`, `git:owner/repo`, volledige `https://`-, `ssh://`-, `git://`-, `file://`- en `git@host:owner/repo.git`-kloon-URL's. Voeg `@<ref>` of `#<ref>` toe om vóór de installatie een branch, tag of commit uit te checken.

    Git-installaties klonen naar een tijdelijke map, checken indien aanwezig de gevraagde ref uit en gebruiken vervolgens het normale installatieprogramma voor Plugin-mappen. Daardoor werken manifestvalidatie, installatiebeleid van de operator, installatiewerk van de pakketbeheerder en installatierecords hetzelfde als bij npm-installaties. Vastgelegde git-installaties bevatten de bron-URL/ref plus de opgeloste commit, zodat `openclaw plugins update` de bron later opnieuw kan oplossen.

    Gebruik na installatie vanuit git `openclaw plugins inspect <id> --runtime --json` om runtimeregistraties zoals Gateway-methoden en CLI-opdrachten te verifiëren. Als de Plugin met `api.registerCli` een CLI-hoofdopdracht heeft geregistreerd, voer je die opdracht rechtstreeks uit via de hoofd-CLI van OpenClaw, bijvoorbeeld `openclaw demo-plugin ping`.

  </Accordion>
  <Accordion title="Archieven">
    Ondersteunde archieven: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Systeemeigen OpenClaw Plugin-archieven moeten een geldige `openclaw.plugin.json` bevatten in de hoofdmap van de uitgepakte Plugin; archieven die alleen `package.json` bevatten, worden geweigerd voordat OpenClaw installatierecords schrijft.

    Gebruik `npm-pack:<path.tgz>` wanneer het bestand een npm-pack-tarball is en je
    hetzelfde beheerde npm-projectpad per Plugin wilt gebruiken als bij registerinstallaties,
    inclusief verificatie van `package-lock.json`, scannen van omhooggehesen afhankelijkheden
    en npm-installatierecords. Gewone archiefpaden worden nog steeds als lokale
    archieven geïnstalleerd onder de Plugin-extensiehoofdmap.

    Installaties vanuit Claude-marketplaces worden ook ondersteund.

  </Accordion>
</AccordionGroup>

ClawHub-installaties gebruiken een expliciete `clawhub:<package>`-locator:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

Kale, voor npm geschikte Plugin-specificaties worden tijdens de overgang bij lancering standaard vanuit npm geïnstalleerd, tenzij ze overeenkomen met een officiële Plugin-id:

```bash
openclaw plugins install openclaw-codex-app-server
```

Gebruik `npm:` om uitsluitend npm-resolutie expliciet te maken:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw controleert vóór de installatie de geadverteerde compatibiliteit met de Plugin-API/minimale Gateway-versie. Wanneer de geselecteerde ClawHub-versie een ClawPack-artefact publiceert, downloadt OpenClaw de geversioneerde npm-pack-`.tgz`, verifieert het de ClawHub-digestheader en de artefactdigest en installeert het deze vervolgens via het normale archiefpad. Oudere ClawHub-versies zonder ClawPack-metadata worden nog steeds geïnstalleerd via het oudere verificatiepad voor pakketarchieven. Vastgelegde installaties bewaren hun ClawHub-bronmetadata, artefacttype, npm-integriteit, npm-shasum, tarballnaam en ClawPack-digestgegevens voor latere updates.
ClawHub-installaties zonder versie behouden een vastgelegde specificatie zonder versie, zodat `openclaw plugins update` nieuwere ClawHub-releases kan volgen; expliciete versie- of tagselectoren zoals `clawhub:pkg@1.2.3` en `clawhub:pkg@beta` blijven aan die selector vastgezet.

### Marketplace-verkorte notatie

Gebruik de verkorte notatie `plugin@marketplace` wanneer de marketplacenaam bestaat in Claude's lokale registercache op `~/.claude/plugins/known_marketplaces.json`:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Gebruik `--marketplace` om de marketplacebron expliciet door te geven:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="Marketplacebronnen">
    - een bij Claude bekende marketplacenaam uit `~/.claude/plugins/known_marketplaces.json`
    - een lokale marketplacehoofdmap of `marketplace.json`-pad
    - een verkorte notatie voor een GitHub-repository, zoals `owner/repo`
    - een URL van een GitHub-repository, zoals `https://github.com/owner/repo`
    - een git-URL

  </Tab>
  <Tab title="Regels voor externe marketplaces">
    Voor externe marketplaces die vanuit GitHub of git worden geladen, moeten Plugin-items binnen de gekloonde marketplace-repository blijven. OpenClaw accepteert relatieve padbronnen uit die repository en weigert HTTP(S)-, absolute-pad-, git-, GitHub- en andere niet-padgebonden Plugin-bronnen uit externe manifesten.
  </Tab>
</Tabs>

Voor lokale paden en archieven detecteert OpenClaw automatisch:

- systeemeigen OpenClaw-Plugins (`openclaw.plugin.json`)
- Codex-compatibele bundels (`.codex-plugin/plugin.json`)
- Claude-compatibele bundels (`.claude-plugin/plugin.json`, of de standaardindeling van Claude-componenten wanneer dat manifestbestand ontbreekt)
- Cursor-compatibele bundels (`.cursor-plugin/plugin.json`)

Beheerde lokale installaties moeten Plugin-mappen of archieven zijn. Losstaande
`.js`-, `.mjs`-, `.cjs`- en `.ts`-Pluginbestanden worden door
`plugins install` niet naar de beheerde Plugin-hoofdmap gekopieerd en evenmin geladen door ze rechtstreeks in
`~/.openclaw/extensions` of `<workspace>/.openclaw/extensions` te plaatsen; deze
automatisch ontdekte hoofdmappen laden Plugin-pakket- of bundelmappen en slaan
scripts op het hoogste niveau over als lokale helpers. Neem losstaande bestanden in plaats daarvan expliciet op in
`plugins.load.paths`.

<Note>
Compatibele bundels worden in de normale Plugin-hoofdmap geïnstalleerd en nemen deel aan dezelfde workflow voor weergeven/informatie/inschakelen/uitschakelen. Momenteel worden Skills in bundels, Claude-opdracht-Skills, standaardwaarden voor Claude-`settings.json`, standaardwaarden voor Claude-`.lsp.json` / in het manifest gedeclareerde `lspServers` en Cursor-opdracht-Skills en compatibele Codex-hookmappen ondersteund; andere gedetecteerde bundelmogelijkheden worden weergegeven in diagnostiek/informatie, maar zijn nog niet gekoppeld aan runtime-uitvoering.
</Note>

Gebruik `-l`/`--link` om naar een lokale Plugin-map te verwijzen zonder deze te kopiëren (voegt
toe aan `plugins.load.paths`):

```bash
openclaw plugins install -l ./my-plugin
```

`--link` wordt niet ondersteund bij `--marketplace`- of `git:`-installaties en
vereist een lokaal pad dat al bestaat. Geef voor een niet-interactieve lokale koppeling
`--force` door nadat je de bron hebt beoordeeld; hiermee wordt de herkomst bevestigd, maar de
gekoppelde map wordt niet gekopieerd of overschreven.

<Note>
Plugins uit een workspace die vanuit de extensiehoofdmap van een workspace worden ontdekt, worden pas
geïmporteerd of uitgevoerd nadat ze expliciet zijn ingeschakeld. Voer voor lokale ontwikkeling
`openclaw plugins enable <plugin-id>` uit of stel
`plugins.entries.<plugin-id>.enabled: true` in; als je configuratie
`plugins.allow` gebruikt, neem je daar ook dezelfde Plugin-id in op. Deze fail-closed-regel
geldt ook wanneer kanaalconfiguratie expliciet is gericht op een Plugin uit een workspace om
deze alleen voor configuratie te laden, zodat lokale configuratiecode van een kanaal-Plugin niet wordt uitgevoerd zolang die
workspace-Plugin uitgeschakeld blijft of van de toelatingslijst is uitgesloten. Gekoppelde installaties
en expliciete `plugins.load.paths`-items volgen het normale beleid voor hun
opgeloste Plugin-herkomst. Zie
[Pluginbeleid configureren](/nl/tools/plugin#configure-plugin-policy)
en [Configuratiereferentie](/nl/gateway/configuration-reference#plugins).

Gebruik `--pin` bij npm-installaties om de exact opgeloste specificatie (`name@version`) op te slaan in de beheerde Plugin-index, terwijl het standaardgedrag niet-vastgezet blijft.
</Note>

## Lijst

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  Alleen ingeschakelde plugins weergeven.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Overschakelen van de tabelweergave naar detailregels per plugin met metadata over indeling/bron/herkomst/versie/activering.
</ParamField>
<ParamField path="--json" type="boolean">
  Machineleesbare inventaris plus registerdiagnostiek en installatiestatus van pakketafhankelijkheden.
</ParamField>

<Note>
`plugins list` leest eerst het persistente lokale pluginregister, met een uitsluitend uit manifesten afgeleide terugvaloptie wanneer het register ontbreekt of ongeldig is. Dit is nuttig om te controleren of een plugin is geïnstalleerd, ingeschakeld en zichtbaar voor de planning van een koude start, maar het is geen live runtimecontrole van een Gateway-proces dat al actief is. Start na wijzigingen aan plugincode, inschakeling, hookbeleid of `plugins.load.paths` de Gateway die het kanaal bedient opnieuw op voordat je verwacht dat nieuwe `register(api)`-code of hooks worden uitgevoerd. Controleer bij externe/containerimplementaties of je het daadwerkelijke onderliggende `openclaw gateway run`-proces opnieuw opstart en niet alleen een wrapperproces.

`plugins list --json` bevat de `dependencyStatus` van elke plugin uit `package.json`
`dependencies` en `optionalDependencies`. OpenClaw controleert of die pakketnamen
aanwezig zijn in het normale Node-opzoekpad `node_modules` van de plugin; het
importeert geen runtimecode van plugins, voert geen pakketbeheerder uit en herstelt
geen ontbrekende afhankelijkheden.
</Note>

Als bij het opstarten `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` wordt gelogd,
voer dan `openclaw plugins list --enabled --verbose` of
`openclaw plugins inspect <id>` uit met een vermelde plugin-id om de plugin-
id's te bevestigen en vertrouwde id's naar `plugins.allow` in `openclaw.json` te kopiëren. Wanneer de
waarschuwing elke ontdekte plugin kan vermelden, wordt een direct te plakken
`plugins.allow`-fragment afgedrukt waarin die id's al zijn opgenomen. Als een plugin wordt geladen
zonder herkomstinformatie over de installatie of het laadpad, inspecteer je die plugin-id en leg je
de vertrouwde id vervolgens vast in `plugins.allow`, of installeer je de plugin opnieuw vanuit een vertrouwde bron
zodat OpenClaw de installatieherkomst vastlegt.

Voor werk aan gebundelde plugins in een verpakte Docker-image koppel je de
bronmap van de plugin via een bind-mount aan het overeenkomende verpakte bronpad, zoals
`/app/extensions/synology-chat`. OpenClaw ontdekt die gekoppelde bronoverlay
vóór `/app/dist/extensions/synology-chat`; een gewoon gekopieerde bronmap
blijft inactief, zodat normale verpakte installaties nog steeds de gecompileerde dist gebruiken.

Voor foutopsporing van runtimehooks:

- `openclaw plugins inspect <id> --runtime --json` toont geregistreerde hooks en diagnostiek uit een inspectiepass waarbij de module wordt geladen. Runtime-inspectie installeert nooit afhankelijkheden; gebruik `openclaw doctor --fix` om verouderde afhankelijkheidsstatus op te schonen of ontbrekende downloadbare plugins te herstellen waarnaar de configuratie verwijst.
- `openclaw gateway status --deep --require-rpc` bevestigt de bereikbare Gateway-URL/het profiel, aanwijzingen voor service/proces, het configuratiepad en de RPC-status.
- Niet-gebundelde gesprekshooks (`llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, `agent_end`) vereisen `plugins.entries.<id>.hooks.allowConversationAccess=true`.

### Pluginindex

Metadata over plugininstallaties is machinaal beheerde status, geen gebruikersconfiguratie. Installaties en updates schrijven deze naar de gedeelde SQLite-statusdatabase onder de actieve OpenClaw-statusmap. De rij `installed_plugin_index` slaat duurzame `installRecords`-metadata op, waaronder records voor defecte of ontbrekende pluginmanifesten, plus een uit manifesten afgeleide cache voor het koude register die wordt gebruikt door `openclaw plugins update`, verwijdering, diagnostiek en het koude pluginregister.

`plugins.installs` is een uitgefaseerd configuratieoppervlak voor handmatige configuratie. Runtime- en updateopdrachten lezen uitsluitend de SQLite-index van geïnstalleerde plugins. Voer `openclaw doctor --fix` uit om verouderde configuratierecords in de index te importeren en de uitgefaseerde sleutel te verwijderen voordat je de normale runtime gebruikt.

## Verwijderen

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall` verwijdert pluginrecords uit `plugins.entries`, de persistente pluginindex, vermeldingen in lijsten met toegestane/geweigerde plugins en, indien van toepassing, gekoppelde `plugins.load.paths`-vermeldingen. Tenzij `--keep-files` is ingesteld, verwijdert de verwijdering ook de bijgehouden beheerde installatiemap, maar alleen wanneer deze binnen de hoofdmap voor pluginextensies van OpenClaw wordt herleid. Als de plugin momenteel eigenaar is van het `memory`- of `contextEngine`-slot, wordt dat slot teruggezet op de standaardwaarde (`memory-core` voor geheugen, `legacy` voor de contextengine).

`uninstall` drukt een voorbeeld af van wat wordt verwijderd en toont vervolgens de vraag `Uninstall plugin "<id>"?` voordat wijzigingen worden aangebracht. Geef `--force` door om de bevestigingsvraag over te slaan (nuttig voor scripts en niet-interactieve uitvoeringen); zonder deze optie vereist de verwijdering een interactieve TTY. `--dry-run` drukt hetzelfde voorbeeld af en sluit af zonder een vraag te stellen of iets te wijzigen.

<Note>
`--keep-config` wordt ondersteund als een verouderde alias voor `--keep-files`.
</Note>

## Bijwerken

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

Updates zijn van toepassing op bijgehouden plugininstallaties in de beheerde pluginindex en op bijgehouden hookpakketinstallaties in de gedeelde SQLite-status. Ze gebruiken opnieuw de bron die de gebruiker al heeft gekozen bij het installeren van de plugin, zodat geen tweede bronbevestiging nodig is.

<AccordionGroup>
  <Accordion title="Plugin-id tegenover npm-specificatie herleiden">
    Wanneer je een plugin-id doorgeeft, gebruikt OpenClaw de vastgelegde installatiespecificatie voor die plugin opnieuw. Dit betekent dat eerder opgeslagen dist-tags zoals `@beta` en exact vastgelegde versies ook bij latere uitvoeringen van `update <id>` worden gebruikt.

    Tijdens `update <id> --dry-run` blijven exacte npm-installaties vastgelegd. Als OpenClaw ook de standaardreeks van het pakketregister kan herleiden en die standaardreeks nieuwer is dan de geïnstalleerde vastgelegde versie, meldt de proefuitvoering de vastlegging en drukt deze de expliciete pakketupdateopdracht `@latest` af om de standaardreeks van het register te volgen.

    Die regel voor gerichte updates verschilt van het bulkonderhoudspad `openclaw plugins update --all`. Bulkuppdates respecteren nog steeds gewone bijgehouden installatiespecificaties, maar vertrouwde officiële OpenClaw-pluginrecords kunnen worden gesynchroniseerd met het huidige doel uit de officiële catalogus in plaats van op een verouderd exact officieel pakket te blijven. Gebruik een gerichte `update <id>` wanneer je opzettelijk een exacte of getagde officiële specificatie ongewijzigd wilt laten.

    Voor npm-installaties kun je ook een expliciete npm-pakketspecificatie met een dist-tag of exacte versie doorgeven. OpenClaw herleidt die pakketnaam naar het bijgehouden pluginrecord, werkt die geïnstalleerde plugin bij en legt de nieuwe npm-specificatie vast voor toekomstige updates op basis van de id.

    Als je de npm-pakketnaam zonder versie of tag doorgeeft, wordt deze eveneens naar het bijgehouden pluginrecord herleid. Gebruik dit wanneer een plugin op een exacte versie was vastgelegd en je deze wilt terugzetten op de standaardreleasereeks van het register.

  </Accordion>
  <Accordion title="Updates voor het bètakanaal">
    Een gerichte `openclaw plugins update <id-or-npm-spec>` gebruikt de bijgehouden pluginspecificatie opnieuw, tenzij je een nieuwe specificatie doorgeeft. Een bulkbewerking met `openclaw plugins update --all` gebruikt de geconfigureerde `update.channel` wanneer vertrouwde officiële pluginrecords worden gesynchroniseerd met het doel uit de officiële catalogus, zodat installaties van het bètakanaal op de bètareleasereeks kunnen blijven in plaats van stilzwijgend naar stable/latest te worden genormaliseerd.

    `openclaw update` kent ook het actieve OpenClaw-updatekanaal: op het bètakanaal proberen npm- en ClawHub-pluginrecords uit de standaardreeks eerst `@beta`. Ze vallen terug op de vastgelegde standaard-/latest-specificatie als er geen bètarelease van de plugin bestaat; npm-plugins vallen ook terug wanneer het bètapakket bestaat maar niet slaagt voor de installatievalidatie. Die terugval wordt als waarschuwing gemeld en laat de kernupdate niet mislukken. Exacte versies en expliciete tags blijven voor gerichte updates op die selector vastgelegd.

  </Accordion>
  <Accordion title="Versiecontroles en integriteitsafwijkingen">
    Vóór een live npm-update controleert OpenClaw de geïnstalleerde pakketversie aan de hand van de metadata van het npm-register. Als de geïnstalleerde versie en de vastgelegde artefactidentiteit al overeenkomen met het herleide doel, wordt de update overgeslagen zonder `openclaw.json` te downloaden, opnieuw te installeren of te herschrijven.

    Wanneer een opgeslagen integriteitshash bestaat en de hash van het opgehaalde artefact verandert, behandelt OpenClaw dit als een afwijking van het npm-artefact. De interactieve opdracht `openclaw plugins update` drukt de verwachte en werkelijke hashes af en vraagt om bevestiging voordat wordt doorgegaan. Niet-interactieve updatehulpmiddelen stoppen standaard veilig, tenzij de aanroeper een expliciet vervolgbeleid opgeeft.

  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install bij bijwerken">
    `--dangerously-force-unsafe-install` wordt voor compatibiliteit ook geaccepteerd bij `plugins update`, maar is verouderd en verandert het gedrag van pluginupdates niet langer. `security.installPolicy` van de beheerder kan updates nog steeds blokkeren; `before_install`-hooks van plugins zijn alleen van toepassing in processen waarin pluginhooks zijn geladen.
  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk bij bijwerken">
    Updates van communityplugins die door ClawHub worden ondersteund, voeren vóór het downloaden van het vervangende pakket dezelfde vertrouwenscontrole voor de exacte release uit als installaties. Gebruik `--acknowledge-clawhub-risk` voor gecontroleerde automatisering die moet doorgaan wanneer de geselecteerde ClawHub-release een riskante vertrouwenswaarschuwing heeft. Officiële ClawHub-pakketten en gebundelde OpenClaw-pluginbronnen slaan deze vraag over releasevertrouwen over.
  </Accordion>
</AccordionGroup>

## Inspecteren

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

Inspectie toont identiteit, laadstatus, bron, manifestmogelijkheden, beleidsvlaggen, diagnostiek, installatiemetadata, bundelmogelijkheden en eventueel gedetecteerde ondersteuning voor MCP- of LSP-servers, zonder standaard de runtime van de plugin te importeren. JSON-uitvoer bevat de contracten van het pluginmanifest, zoals `contracts.agentToolResultMiddleware` en `contracts.trustedToolPolicies`, zodat beheerders verklaringen over vertrouwde oppervlakken kunnen controleren voordat ze een plugin inschakelen of opnieuw opstarten. Voeg `--runtime` toe om de pluginmodule te laden en geregistreerde hooks, hulpmiddelen, opdrachten, services, gatewaymethoden en HTTP-routes op te nemen. Runtime-inspectie meldt ontbrekende pluginafhankelijkheden rechtstreeks; installaties en reparaties blijven in `openclaw plugins install`, `openclaw plugins update` en `openclaw doctor --fix`.

CLI-opdrachten waarvan een plugin eigenaar is, worden doorgaans als hoofdopdrachtgroepen `openclaw` geïnstalleerd, maar plugins kunnen ook geneste opdrachten registreren onder een kernbovenliggend element zoals `openclaw nodes`. Nadat `inspect --runtime` een opdracht onder `cliCommands` toont, voer je deze uit via het vermelde pad; een plugin die bijvoorbeeld `demo-git` registreert, kan worden geverifieerd met `openclaw demo-git ping`.

Elke plugin wordt geclassificeerd op basis van wat deze daadwerkelijk tijdens runtime registreert:

| Vorm                | Betekenis                                                         |
| ------------------- | ----------------------------------------------------------------- |
| `plain-capability`  | precies één type mogelijkheid (bijv. een plugin met alleen een provider) |
| `hybrid-capability` | meer dan één type mogelijkheid (bijv. tekst + spraak + afbeeldingen) |
| `hook-only`         | alleen hooks, zonder mogelijkheden, hulpmiddelen, opdrachten, services of routes |
| `non-capability`    | hulpmiddelen/opdrachten/services, maar geen mogelijkheden          |

Zie [Pluginvormen](/nl/plugins/architecture#plugin-shapes) voor meer informatie over het mogelijkhedenmodel.

<Note>
De vlag `--json` voert een machineleesbaar rapport uit dat geschikt is voor scripts en audits. `inspect --all` geeft een tabel voor de hele vloot weer met kolommen voor vorm, soorten mogelijkheden, compatibiliteitsmeldingen, bundelmogelijkheden en een samenvatting van hooks. `info` is een alias voor `inspect`.
</Note>

## Doctor

```bash
openclaw plugins doctor
```

`doctor` rapporteert fouten bij het laden van plugins, diagnostiek voor manifesten/detectie, compatibiliteitsmeldingen en verwijzingen naar verouderde pluginconfiguratie, zoals ontbrekende pluginslots. Wanneer de installatieboom en pluginconfiguratie schoon zijn, wordt `No plugin issues detected.` weergegeven. Als er verouderde configuratie resteert, maar de installatieboom verder in orde is, vermeldt de samenvatting dit in plaats van te suggereren dat alle plugins volledig in orde zijn.

Als een geconfigureerde plugin op schijf aanwezig is, maar wordt geblokkeerd door de padveiligheidscontroles van de loader, behoudt de configuratievalidatie de pluginvermelding en rapporteert deze als `present but blocked`. Verhelp de voorafgaande diagnostische melding over de geblokkeerde plugin, bijvoorbeeld over padeigendom of wereldwijd beschrijfbare machtigingen, in plaats van de configuratie voor `plugins.entries.<id>` of `plugins.allow` te verwijderen.

Voer bij fouten in de modulestructuur, zoals ontbrekende exports van `register`/`activate`, de opdracht opnieuw uit met `OPENCLAW_PLUGIN_LOAD_DEBUG=1` om een compacte samenvatting van de exportstructuur in de diagnostische uitvoer op te nemen.

## Register

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

Het lokale pluginregister is OpenClaws permanente model voor cold reads van de identiteit, inschakeling, bronmetadata en het eigendom van bijdragen van geïnstalleerde plugins. Normaal opstarten, het opzoeken van providereigenaren, de classificatie van kanaalconfiguratie en de plugininventaris kunnen dit lezen zonder runtime-modules van plugins te importeren.

Gebruik `plugins registry` om te controleren of het permanente register aanwezig, actueel of verouderd is. Gebruik `--refresh` om het opnieuw op te bouwen vanuit de permanente pluginindex, het configuratiebeleid en de manifest-/pakketmetadata. Dit is een herstelpad, geen pad voor runtime-activering.

`openclaw doctor --fix` herstelt ook afwijkingen in beheerde npm-installaties die aan het register grenzen. Als een verweesd of hersteld `@openclaw/*`-pakket onder een beheerd npm-project voor plugins of de oude platte beheerde npm-hoofdmap een meegeleverde plugin overschaduwt, verwijdert Doctor dat verouderde pakket en bouwt het register opnieuw op, zodat bij het opstarten tegen het meegeleverde manifest wordt gevalideerd. Wanneer een gezaghebbende installatierecord één beheerde generatie selecteert, maar oudere platte mappen of generatiemappen achterblijven, stelt Doctor die verouderde bomen buiten gebruik, zodat ze kunnen worden opgeschoond nadat de Gateway opnieuw is gestart. Doctor koppelt ook het `openclaw`-pakket van de host opnieuw aan beheerde npm-plugins die `peerDependencies.openclaw` declareren, zodat pakketlokale runtime-imports zoals `openclaw/plugin-sdk/*` na updates of npm-herstel weer kunnen worden omgezet.

## Marketplace

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries` vermeldt items uit de geconfigureerde OpenClaw-marketplacefeed. Standaard wordt geprobeerd de gehoste feed te gebruiken, met als terugvaloptie de laatst geaccepteerde momentopname of meegeleverde gegevens. Gebruik `--feed-profile <name>` om een specifiek geconfigureerd profiel te lezen, `--feed-url <url>` om een expliciete URL van een gehoste feed te lezen en `--offline` om de laatst geaccepteerde momentopname te lezen zonder de feed op te halen.

`plugins marketplace refresh` vernieuwt de geconfigureerde momentopname van de gehoste feed en rapporteert of OpenClaw gehoste gegevens, een gehoste momentopname of meegeleverde terugvalgegevens heeft geaccepteerd. Gebruik `--expected-sha256` wanneer een aanroeper vereist dat de opdracht mislukt tenzij een nieuwe gehoste payload overeenkomt met een vastgezette controlesom.

Marketplace `list` accepteert een lokaal marketplacepad, een `marketplace.json`-pad, een GitHub-verkorting zoals `owner/repo`, een URL van een GitHub-repository of een git-URL. `--json` geeft het vastgestelde bronlabel weer, samen met het geparseerde marketplacemanifest en de pluginitems.

Marketplace-vernieuwing laadt een gehoste OpenClaw-marketplacefeed en slaat het
gevalideerde antwoord permanent op als de lokale momentopname van de gehoste feed. Zonder opties wordt
het geconfigureerde standaardfeedprofiel gebruikt. Gebruik `--feed-profile <name>` om een
specifiek geconfigureerd profiel te vernieuwen, `--feed-url <url>` om een expliciete gehoste
feed-URL te vernieuwen, `--expected-sha256 <sha256>` om een overeenkomende controlesom van de payload te vereisen
(`sha256:<hex>` of een kale hexadecimale digest van 64 tekens) en `--json` voor
machineleesbare uitvoer. Expliciete URL's van gehoste feeds mogen geen
referenties, queryreeksen of fragmenten bevatten. Vernieuwingen zonder vastgezette controlesom kunnen een
gehoste momentopname of meegeleverd terugvalresultaat rapporteren zonder dat de opdracht mislukt. Vastgezette
vernieuwingen mislukken tenzij ze een nieuwe gehoste payload accepteren, en geslaagde gehoste
vernieuwingen mislukken als OpenClaw de gevalideerde momentopname niet permanent kan opslaan.

Het ingebouwde profiel `clawhub-public` verwacht payloadidentiteit
`clawhub-official`. OpenClaw zal de openbare productiesleutel van ClawHub meeleveren nadat
ClawHub die sleutel heeft gegenereerd en overgedragen. Tot die tijd verleent het ingebouwde profiel
geen installatiebevoegdheid op basis van ondertekende feeds. Openbare sleutels moeten afkomstig zijn van een vertrouwd
release- of beheerderskanaal, niet van een sleuteleindpunt op de feedhost.

OpenClaw verifieert de DSSE-envelop en vereist, wanneer een profiel `feedId`
declareert, dat de gedecodeerde payload-ID daarmee overeenkomt. Het ingebouwde profiel `clawhub-public`
declareert altijd zijn identiteit, waardoor wordt voorkomen dat een geldig document voor een andere
feed via dat profiel opnieuw wordt afgespeeld.

Tijdens de gefaseerde uitrol behouden bestaande aangepaste ondertekende profielen die `feedId`
weglaten de verificatie van handtekeningen zonder koppeling aan de payloadidentiteit. Nieuwe aangepaste
profielen moeten `feedId` declareren. Het configuratieoppervlak voor feedprofielen wordt
afzonderlijk toegevoegd, samen met de presentatiemetadata die Control UI nodig heeft; de
diagnostische melding van Doctor moet de beheerder vragen een ontbrekende identiteit op te geven en mag
er geen afleiden uit de feed-URL. Deze vertrouwenskoppeling herstelt de buiten gebruik gestelde
hoofdwerksleutel `marketplaces` niet.

## Gerelateerd

- [Plugins bouwen](/nl/plugins/building-plugins)
- [CLI-referentie](/nl/cli)
- [ClawHub](/clawhub)
