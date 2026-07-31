---
read_when:
    - Een skill of plugin publiceren
    - Fouten met eigenaar- of pakketbereik opsporen
    - Publicatiegedrag voor UI, CLI of backend toevoegen
summary: Hoe publiceren via ClawHub werkt voor skills, plugins, eigenaren, scopes, releases en reviews.
x-i18n:
    generated_at: "2026-07-27T05:45:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 582dffaf4429e9f24d7c38f2809cc7dc05f8471e4ae2f9c6be60153cc8604e3f
    source_path: clawhub/publishing.md
    workflow: 16
---

# Publiceren

Bij publiceren wordt een Skills-map of Plugin-pakket naar ClawHub verzonden onder de eigenaar die je
kiest. ClawHub controleert of je token voor die eigenaar mag publiceren, valideert de
metadata, naam, versie, bestanden en broninformatie, slaat vervolgens de release op
en start geautomatiseerde beveiligingscontroles.

Als de validatie mislukt, wordt er niets gepubliceerd. Nieuwe releases blijven mogelijk ook buiten de
normale installatie- en downloadmogelijkheden totdat de beoordeling is afgerond.

## Skills

De eenvoudigste manier om te publiceren is via de CLI. Meld je aan en publiceer vervolgens een lokale Skills-map:

```bash
clawhub login
clawhub skill publish ./my-skill \
  --slug my-skill \
  --name "Mijn Skill" \
  --owner <owner>
```

Gebruik `--owner <handle>` wanneer je publiceert onder een organisatie-eigenaar. Laat dit weg om als
de geauthenticeerde gebruiker te publiceren. Bij het publiceren wordt ongewijzigde inhoud overgeslagen. Een nieuwe Skill begint
bij `1.0.0` en bij latere wijzigingen wordt automatisch de volgende patchversie gepubliceerd. Geef
`--version` alleen door wanneer je een expliciete versie nodig hebt.

Gebruik voor catalogusrepository's de herbruikbare
[`skill-publish.yml`-workflow](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml) van ClawHub.
Deze roept `skill publish` aan voor elke directe Skills-map onder `root` (standaard:
`skills`), of alleen voor de map die als `skill_path` is opgegeven.

```yaml
jobs:
  publish:
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@main
    with:
      owner: <owner>
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Gebruik `dry_run: true` om een voorbeeld van nieuwe en gewijzigde Skills te bekijken zonder deze te publiceren.

## Plugins

Plugins gebruiken pakketnamen in npm-stijl. Pakketnamen met een scope bevatten de eigenaar in
het eerste deel van de naam:

```text
@owner/package-name
```

De scope moet overeenkomen met de geselecteerde publicatie-eigenaar. Als je pakket
`@openclaw/dronzer` heet, kan het alleen als `@openclaw` worden gepubliceerd. Als je als
`@vintageayu` publiceert, wijzig je de pakketnaam in `@vintageayu/dronzer`.

Dit voorkomt dat een pakket aanspraak maakt op de naamruimte van een organisatie waarover de publiceerder
geen controle heeft.

Als je de rechtmatige eigenaar bent van een organisatie, merk, pakketscope, eigenaarshandle of
naamruimte die al op ClawHub is geclaimd of gereserveerd, open je een
[issue voor een claim op een organisatie/naamruimte](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
met openbaar, niet-gevoelig bewijs. Zie
[Claims op organisaties en naamruimten](/clawhub/namespace-claims) voor wat je moet opnemen en wat je
buiten openbare issues moet houden.

### Voordat je een Plugin publiceert

- Kies een eigenaar die overeenkomt met de pakketscope.
- Neem `openclaw.plugin.json` op. Code-Plugins hebben ook `package.json` nodig met
  `openclaw.compat.pluginApi` en `openclaw.build.openclawVersion`.
- Als je een aangepast pictogram voor de Plugincatalogus op de startpagina en Pluginlijstpagina's wilt tonen,
  voeg je `icon` toe aan `openclaw.plugin.json` met een willekeurige HTTPS-afbeeldings-URL.
- Neem de bronrepository en metadata van de exacte commit op, of gebruik de CLI vanuit een
  door GitHub ondersteunde checkout zodat deze de gegevens kan detecteren.
- Voer `clawhub package validate <source>` uit voordat je publiceert. Zie voor bevindingen over pakketten,
  manifesten, SDK-imports of artefacten
  [Oplossingen voor Pluginvalidatie](/clawhub/plugin-validation-fixes).
- Voer `clawhub package publish <source> --dry-run` uit voordat je een release maakt.
- Houd er rekening mee dat nieuwe releases buiten de openbare installatiemogelijkheden blijven totdat de geautomatiseerde
  beveiligingscontroles en verificatie zijn afgerond.

### Vertrouwd publiceren voor pakketten

Vertrouwd publiceren van pakketten vereist twee stappen:

1. Publiceer het pakket eenmaal via de normale handmatige of met een token geauthenticeerde
   `clawhub package publish`. Hiermee wordt de pakketrij gemaakt en worden de
   pakketbeheerders vastgesteld die de configuratie van de vertrouwde publiceerder mogen wijzigen.
2. Een pakketbeheerder stelt de configuratie van de vertrouwde GitHub Actions-publiceerder in:

```bash
clawhub package trusted-publisher set @owner/package-name \
  --repository owner/repo \
  --workflow-filename package-publish.yml
```

Nadat de configuratie is ingesteld, kunnen toekomstige ondersteunde publicaties via GitHub Actions
OIDC/vertrouwd publiceren gebruiken zonder een langlevend ClawHub-token in de
repository op te slaan. De geconfigureerde repository en workflowbestandsnaam moeten overeenkomen met de
OIDC-claim van GitHub Actions. Als je ook `--environment <name>` doorgeeft, moet de
omgevingsclaim van GitHub Actions exact met die naam overeenkomen.

ClawHub verifieert de geconfigureerde GitHub-repository wanneer de configuratie van de vertrouwde publiceerder
wordt ingesteld. Openbare repository's kunnen via openbare GitHub-metadata worden geverifieerd.
Voor privérepository's moet ClawHub toegang hebben tot die GitHub-repository,
bijvoorbeeld via een toekomstige installatie van de ClawHub GitHub App of een andere
geautoriseerde GitHub-integratie.

De huidige herbruikbare workflow voor het publiceren van pakketten ondersteunt vertrouwd publiceren zonder geheimen
voor `workflow_dispatch`-publicaties wanneer `id-token: write`
beschikbaar is. Voor echte publicaties via een tagpush is `clawhub_token` nog steeds vereist, dus houd
`CLAWHUB_TOKEN` beschikbaar voor tagreleases, eerste publicaties, niet-vertrouwde pakketten
of noodpublicaties.

Bekijk of verwijder de configuratie met:

```bash
clawhub package trusted-publisher get @owner/package-name
clawhub package trusted-publisher delete @owner/package-name
```

Het verwijderen van de configuratie van de vertrouwde publiceerder is de terugvalroute. Hierdoor wordt het toekomstig
uitgeven van tokens voor vertrouwd publiceren uitgeschakeld totdat een pakketbeheerder de configuratie opnieuw instelt.

## Veelgestelde vragen

### Pakketscope moet overeenkomen met de geselecteerde eigenaar

Als de pakketscope en de geselecteerde eigenaar niet overeenkomen, weigert ClawHub de
publicatie:

```text
Pakketscope "@openclaw" moet overeenkomen met de geselecteerde eigenaar "@vintageayu".
Publiceer als "@openclaw" of wijzig de naam van dit pakket in "@vintageayu/dronzer".
```

Om dit op te lossen, kies je de eigenaar die door de pakketscope wordt genoemd, of wijzig je de naam van het
pakket zodat de scope overeenkomt met de eigenaar waaronder je kunt publiceren.

Als de pakketnaam al de juiste scope heeft, maar het pakket eigendom is van de
verkeerde publiceerder, draag je in plaats daarvan het eigendom over:

```sh
clawhub package transfer @opik/opik-openclaw --to opik
```

Gebruik overdracht van een pakket of Skill alleen wanneer je beheerderstoegang hebt tot zowel de
huidige eigenaar als de bestemmingspubliceerder. Met pakketoverdracht kun je niet
publiceren in een scope die je niet kunt beheren.

Als je geen toegang hebt tot de huidige eigenaar, maar denkt dat jouw organisatie, project of
merk de rechtmatige eigenaar van de naamruimte is, open je een
[issue voor een claim op een organisatie/naamruimte](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
met openbaar, niet-gevoelig bewijs voor beoordeling door medewerkers. Zie
[Claims op organisaties en naamruimten](/clawhub/namespace-claims) voordat je het issue indient.

Dit beschermt de naamruimten van organisaties. Een pakket met de naam `@openclaw/dronzer` claimt de
naamruimte `@openclaw`, zodat alleen publiceerders met toegang tot de eigenaar `@openclaw`
het kunnen publiceren.
