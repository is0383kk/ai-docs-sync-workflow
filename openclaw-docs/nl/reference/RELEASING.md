---
doc-schema-version: 1
read_when:
    - Op zoek naar definities van openbare releasekanalen
    - Releasevalidatie of pakketacceptatie uitvoeren
    - Op zoek naar versienamen en releasefrequentie
summary: Releasekanalen, checklist voor operators, validatieomgevingen, versienaamgeving en ritme
title: Releasebeleid
x-i18n:
    generated_at: "2026-07-27T06:08:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw biedt vier updatekanalen voor gebruikers:

- stable: de gepromoveerde reguliere release op npm `latest`
- extended-stable: de `.33+`-onderhoudslijn van de laatst voltooide maand op
  npm `extended-stable`
- beta: prereleasetags op npm `beta`
- dev: de voortschrijdende head van `main`

Extended-stable levert de Gateway, officiële npm-plugins en
Docker-images van de voorgaande maand zonder de reguliere selectors `latest` of `main` te verplaatsen.

Tideclaw-alfabuilds vormen een afzonderlijk intern prereleasetraject (npm-dist-tag `alpha`), beschreven onder [NPM-workflowinvoer](#npm-workflow-inputs) en [Testboxen voor releases](#release-test-boxes).

## Versienamen

- Maandelijkse extended-stable-releaseversie van de Gateway: `YYYY.M.PATCH`, met `PATCH >= 33`, git-tag `vYYYY.M.PATCH`
- Dagelijkse/reguliere definitieve releaseversie: `YYYY.M.PATCH`, met `PATCH < 33`, git-tag `vYYYY.M.PATCH`
- Reguliere terugvalcorrectiereleaseversie: `YYYY.M.PATCH-N`, git-tag `vYYYY.M.PATCH-N`
- Bèta-prereleaseversie: `YYYY.M.PATCH-beta.N`, git-tag `vYYYY.M.PATCH-beta.N`
- Alfa-prereleaseversie: `YYYY.M.PATCH-alpha.N`, git-tag `vYYYY.M.PATCH-alpha.N`
- Vul maand of patch nooit met voorloopnullen aan
- `PATCH` is een opeenvolgend nummer van de maandelijkse releasetrein, geen kalenderdag. Reguliere definitieve en bètareleases schuiven de huidige trein door; tags die uitsluitend voor alfa zijn, gebruiken of verhogen het bèta-/reguliere patchnummer nooit. Negeer daarom verouderde tags die uitsluitend voor alfa zijn en hogere patchnummers hebben bij het selecteren van een bèta- of reguliere trein.
- Alfa-/nightly-builds gebruiken de volgende nog niet uitgebrachte patchtrein en verhogen bij herhaalde builds uitsluitend `alpha.N`. Zodra die patch een bèta heeft, gaan nieuwe alfabuilds naar de daaropvolgende patch.
- npm-versies zijn onveranderlijk: verwijder, herpubliceer of hergebruik nooit een gepubliceerde tag. Maak in plaats daarvan het volgende prereleasenummer of de volgende maandelijkse patch.
- `latest` blijft de huidige reguliere/dagelijkse npm-lijn volgen; `beta` is het huidige installatiedoel voor bèta
- `extended-stable` betekent de ondersteunde Gateway-distributie van de voorgaande maand, beginnend bij patch `33`; patch `34` en later zijn onderhoudsreleases op die maandelijkse lijn
- Reguliere definitieve en reguliere correctiereleases publiceren standaard naar npm `beta`; releaseoperators kunnen expliciet `latest` als doel instellen of later een gecontroleerde bètabuild promoveren
- Gateway extended-stable publiceert de core, elke officieel via npm publiceerbare plugin
  en de bijbehorende Docker-images met exact dezelfde versie; zie de speciale workflow hieronder.
- Elke reguliere definitieve release levert tegelijkertijd het npm-pakket, de macOS-app, een ondertekende zelfstandige Android-APK en ondertekende Windows Hub-installatieprogramma's. Bètareleases valideren en publiceren normaal gesproken eerst het npm-/pakkettraject; het bouwen, ondertekenen, notarieel bekrachtigen en promoveren van native apps blijft voorbehouden aan reguliere definitieve releases, tenzij dit expliciet wordt aangevraagd.

## Releasefrequentie

- Releases verschijnen eerst als bèta; stable volgt pas nadat de nieuwste bèta is gevalideerd
- Onderhouders maken releases normaal gesproken vanuit een `release/YYYY.M.PATCH`-branch die vanaf de huidige `main` is gemaakt, zodat releasevalidatie en correcties nieuwe ontwikkeling op `main` niet blokkeren
- Als een bètagag is gepusht of gepubliceerd en moet worden gecorrigeerd, maken onderhouders de volgende `-beta.N`-tag in plaats van de oude te verwijderen of opnieuw te maken
- De gedetailleerde releaseprocedure, goedkeuringen, referenties en herstelnotities zijn uitsluitend voor onderhouders

## Maandelijkse publicatie van Gateway extended-stable

Maak voor de voltooide maand `YYYY.M` de branch `extended-stable/YYYY.M.33` en publiceer
`.33+` vanuit die branch. Tag, branch, checkout, pakketversie, preflight en
validatie moeten naar één commit verwijzen. Vóór `.33` moet de beschermde `main`
een definitieve versie van een latere maand onder patch `33` bevatten; latere onderhoudspatches
blijven in aanmerking komen.

### De kandidaat voorbereiden en stabiliseren

Controleer het nog niet gecontroleerde bereik van de hoofdlijn, stem privébeveiligingswerk af, keur een
begrensde set backports goed en land één gecoördineerde PR. Push niet rechtstreeks naar de canonieke
branch.

Stel op de canonieke branch `YYYY.M.P` in, voer `pnpm release:prep` uit en vereis
die versie in elke publiceerbare officiële plugin. Genereer vanuit het goedgekeurde register een volledige
`## YYYY.M.P`-sectie met `### Highlights`,
`### Changes` en `### Fixes` en commit deze, waarbij voor equivalente
backports naar de oorspronkelijk gemergede `main`-PR's wordt verwezen. Preflight weigert een ontbrekende of lege sectie.

Neem de volledige Docker-releasekanaaleenheid van de huidige main over: workflow, promotor,
beleid, gedeelde classificator, tests en workflowvalidatie. GitHub laadt tagworkflows
uit de getagde commit; een onvolledige kopie kan na het bouwen mislukken of
reguliere aliassen verplaatsen. Voer gerichte controles uit.

Zet de volledige SHA van de branchtip vast. Voer vóór het taggen preflight uit op de exacte npm-bytes
en voer Full Release Validation uit voor die SHA:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

De SHA-vorm is uitsluitend voor preflight. Voer validatie uit op de canonieke branch; publicatie
bindt de workflowreferentie, head-/doel-SHA, run-ID en poging. Bewaar beide ID's en
de geslaagde `run_attempt`; weiger bewijs voor `release-ci/*`.

Classificeer fouten voordat je wijzigingen aanbrengt:

- Product: land nog een goedgekeurde backport-PR.
- Tooling voor het vastgezette doel: backport uitsluitend de kleinste compatibiliteitsreparatie die
  het oude product ongewijzigd test.
- Provider, goedkeuring, runner of service: houd de kandidaat ongewijzigd en gebruik
  het begrensde herhaaltraject.

Elke branchwijziging maakt beide poorten ongeldig. Vereis zodra ze slagen dat de tip nog steeds
gelijk is aan `RELEASE_SHA` en push vervolgens de ondertekende `vYYYY.M.P`. Latere wijzigingen vereisen de volgende
patch; verplaats of verwijder de tag nooit. De push ervan start `Docker Release`.

### De npm-pakketten publiceren

Publiceer elke officieel via npm publiceerbare plugin vanuit dezelfde SHA en bewaar het
ID van de geslaagde run:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

De workflow omvat alle `all-publishable`-pakketten, inclusief ongewijzigde pakketten,
en verifieert elke exacte versie en selector. Herhaalde runs hergebruiken gepubliceerde versies.

Publiceer vervolgens de voorbereide core-tarball met de identiteit van alle drie opgeslagen runs:

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

Voeg uitsluitend voor een niet-productierepetitie
`-f bypass_extended_stable_guard=true` toe aan preflight en publicatie. Dit omzeilt alleen de
maandcontrole, nooit de controles op canonieke referentie, gelijkheid van SHA/tag/versie, herkomst,
goedkeuring of teruglezing. Gebruik dit nooit voor productie.

### Verifiëren en herstellen

Voer vanuit een afzonderlijke schone checkout van de huidige `main`, niet vanuit de vastgezette branch, het volgende uit:

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

Vereis ondertekeningen en npm-herkomst voor de canonieke branch, plus binding van publicatie,
preflight en tarball-digest aan de release-SHA. Beide opdrachten moeten
`YYYY.M.P` retourneren. Verifieer elk voorbereid core-pakket en elke officiële
`all-publishable`-plugin op de exacte versie en selector.

Als alleen de rootselector mislukt, gebruik je de gegenereerde
`npm dist-tag add openclaw@YYYY.M.P extended-stable`-herstelopdracht die in
de workflowsamenvatting wordt weergegeven. Herstel bestaande selectors van plugins of andere voorbereide core-pakketten
via goedgekeurde tooling met geïsoleerde referenties; de OIDC-bron kan
ze niet wijzigen. Publiceer een onveranderlijke versie nooit opnieuw.

Vereis dat `Docker Release` de exacte standaard-, slim-, browser- en architectuurimages
in GHCR en Docker Hub verifieert, inclusief attestaties en platformversies. Deze workflow
mag uitsluitend
`extended-stable`, `extended-stable-slim` en `extended-stable-browser` op basis van
digest doorschuiven; reguliere aliassen blijven ongewijzigd en automatische rollback wordt geweigerd.

Voer voor aliasherstel de door goedkeuring afgeschermde `Docker Channel Promotion` uit vanaf de huidige
`main` met de tag. Deze herhaalt controles van digest, attestatie en platform, staat
een expliciete rollback toe en bouwt images nooit opnieuw.

Slack, Discord en Codex zijn de oorspronkelijk gedocumenteerde ondersteuningsoppervlakken, geen
releasetoelatingslijst: elke officieel via npm publiceerbare plugin wordt geleverd. Alleen de reguliere
controlelijst is verantwoordelijk voor bèta/`latest`, GitHub Releases, ClawHub, native apps, mobiel,
website en privé-dist-tags; voer die stappen niet uit voor dit Gateway-traject.

## Controlelijst voor operators van reguliere releases

Deze controlelijst beschrijft de openbare vorm van het releaseproces. Privéreferenties en details over ondertekening, notariële bekrachtiging, herstel van dist-tags en noodrollbacks blijven in het releasehandboek dat uitsluitend voor onderhouders bestemd is.

1. Begin vanaf de huidige `main`: haal de nieuwste wijzigingen op, bevestig dat de doelcommit is gepusht en bevestig dat de CI van `main` groen genoeg is om er een branch van te maken.
2. Maak `release/YYYY.M.PATCH` vanaf die commit. Backports zijn optioneel; pas uitsluitend de door de operator geselecteerde set toe. Verhoog elke vereiste versielocatie, voer `pnpm release:prep` uit, voltooi releasecorrecties en vereiste forwardports en controleer `src/plugins/compat/registry.ts` plus `src/commands/doctor/shared/deprecation-compat.ts`.
3. Zet de productvolledige commit van vóór de changelog vast als de **Code-SHA**. Voer de deterministische bronpreflight uit en gebruik vervolgens `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`. Hiermee wordt vertrouwde workflowtooling vastgezet, terwijl de volledige Vitest-, Docker-, QA-, pakket- en prestatiematrix exact op de Code-SHA wordt gericht.
4. Classificeer fouten voordat je wijzigingen aanbrengt. Een product-/codefout creëert een nieuwe Code-SHA en vereist een geslaagde volledige validatie voor die SHA. Een fout in workflow, harnas, referenties, goedkeuring of infrastructuur wordt in het verantwoordelijke oppervlak hersteld en opnieuw uitgevoerd voor dezelfde Code-SHA.
5. Genereer pas nadat de Code-SHA groen is de bovenste `CHANGELOG.md`-sectie uit gemergede PR's en rechtstreekse commits sinds de laatste bereikbare uitgebrachte tag. Houd vermeldingen gebruikersgericht en voorkom duplicaten. Wanneer een afwijkende uitgebrachte tag of latere forwardport reeds uitgebrachte PR's opnieuw koppelt, geef je deze expliciet door als `--shipped-ref`.
6. Commit uitsluitend `CHANGELOG.md`. Deze commit is de **Release-SHA**. Het volledige verschil tussen Code-SHA en Release-SHA moet exact `CHANGELOG.md` zijn; elk ander gewijzigd pad brengt de release terug naar stap 2.
7. Voer aan de SHA gebonden Full Release Validation uit voor de Release-SHA met hergebruik van bewijs ingeschakeld. De lichtgewicht parent moet `changelog-only-release-v1` vastleggen, naar de groene Code-SHA verwijzen en geen onderliggende productlanes starten. Hiermee wordt productbewijs hergebruikt; pakketbytes worden niet hergebruikt.
8. Voer `OpenClaw NPM Release` met `preflight_only=true` uit voor de Release-SHA/tag. Bewaar de geslaagde `preflight_run_id`. Hiermee worden de exacte pakketbytes met de definitieve changelog gebouwd en gecontroleerd.
9. Tag de Release-SHA en voer vervolgens de kandidaathulp uit met de geslaagde validatieparent voor de Release-SHA en de npm-preflight, in plaats van een van beide opnieuw te starten:

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   Geef voor stable ook `--windows-node-tag vX.Y.Z` door. De helper verifieert de herkomst van de releaseopmerkingen, de bytes van de npm-preflight, bewijs van installatie/update via Parallels, bewijs van het Telegram-pakket en publicatieplannen voor plugins, en drukt vervolgens de publicatieopdracht af.

   `OpenClaw Release Publish` stuurt de geselecteerde of alle publiceerbare pluginpakketten parallel naar npm en dezelfde set naar ClawHub, en promoveert vervolgens het voorbereide npm-preflightartefact van OpenClaw met de overeenkomende dist-tag zodra de npm-publicatie van de plugins slaagt. De release-checkout blijft de product-/datahoofdmap, terwijl planning en definitieve verificatie worden uitgevoerd vanuit de exacte vertrouwde checkout van de workflowbron, zodat een oudere releasecommit niet ongemerkt verouderde releasetooling kan gebruiken. Voordat een publicatie-subproces wordt gestart, wordt de exacte inhoud van de GitHub-releasepagina gerenderd en gecachet. Wanneer de volledige overeenkomende sectie `CHANGELOG.md` binnen GitHubs limiet van 125,000 tekens en de overeenkomende veiligheidslimiet van 125,000 bytes van de renderer past, bevat de pagina exact die sectie `## YYYY.M.PATCH`, inclusief de kop. Wanneer de bronsectie niet past, behoudt de pagina de exact gegroepeerde redactionele opmerkingen en vervangt deze het te grote bijdragenoverzicht door een stabiele link naar het volledige overzicht in de aan de tag vastgepinde `CHANGELOG.md`; gedeeltelijke overzichten en afgekorte opsommingstekens worden nooit gepubliceerd. De workflow kiest deze volledige of compacte inhoud voordat `### Release verification` wordt toegevoegd; als het bewijsstaartstuk de limiet zou overschrijden, behoudt de workflow de canonieke inhoud en vertrouwt deze in plaats daarvan op het onveranderlijke bijgevoegde bewijs. Stable releases die naar npm `latest` worden gepubliceerd, worden de nieuwste GitHub-release, terwijl stable onderhoudsreleases die op npm `beta` blijven, met GitHub `latest=false` worden gemaakt. De workflow uploadt ook het preflightbewijs van afhankelijkheden, het manifest van de volledige validatie en verificatiebewijs van het register na publicatie naar de GitHub-release voor incidentrespons na de release. De workflow drukt ID's van subprocesruns onmiddellijk af, keurt automatisch release-omgevingspoorten goed die het workflowtoken mag goedkeuren, vat mislukte subtaken samen met de laatste logregels, maakt vooraf de conceptpagina voor de GitHub-release en promoveert Windows- en Android-assets gelijktijdig met de npm-publicatie van OpenClaw, rondt de releasepagina en het afhankelijkheidsbewijs af zodra die fasen slagen, wacht op ClawHub wanneer OpenClaw naar npm wordt gepubliceerd, voert vervolgens de bètaverificatie vanaf de vertrouwde main uit en uploadt bewijs na publicatie voor de GitHub-release, het npm-pakket, geselecteerde npm-pluginpakketten, geselecteerde ClawHub-pakketten, ID's van workflow-subprocesruns en de optionele ID van de NPM Telegram-run. De ClawHub-bootstrapverificatie vereist het exacte vertrouwde workflowpad en de SHA van main, de producerende en terminale runpogingen, de release-SHA, de aangevraagde pakketset, de onveranderlijke pakketartefacttuple en het artefact van de terminale teruglezing uit het register; een geslaagde verouderde run vanaf de releasereferentie wordt niet geaccepteerd.

   Voer daarna de pakketacceptatie na publicatie uit op het gepubliceerde pakket `openclaw@YYYY.M.PATCH-beta.N` of `openclaw@beta`. Als een gepushte of gepubliceerde prerelease moet worden hersteld, maak dan het volgende overeenkomende prereleasenummer; verwijder of herschrijf het oude nooit.

10. Houd bij een mislukte publicatiepoging de Release-SHA ongewijzigd, tenzij de mislukking een defect in het product of de changelog aantoont. Hervat geslaagde onveranderlijke subprocessen en artefacten; bouw of publiceer nooit opnieuw een pakketversie die al is geslaagd.
11. Ga voor stable alleen verder nadat de gecontroleerde bèta of release candidate het vereiste validatiebewijs heeft. De stable npm-publicatie verloopt ook via `OpenClaw Release Publish`, waarbij het geslaagde preflightartefact opnieuw wordt gebruikt via `preflight_run_id`. Gereedheid voor de stable macOS-release vereist ook de verpakte `.zip`, `.dmg`, `.dSYM.zip` en bijgewerkte `appcast.xml` op `main`; de macOS-publicatieworkflow publiceert de ondertekende appcast automatisch naar de openbare `main` nadat de release-assets zijn geverifieerd, of opent/werkt een appcast-PR bij als branchbeveiliging de rechtstreekse push blokkeert. Gereedheid voor stable Windows Hub vereist de ondertekende assets `OpenClawCompanion-Setup-x64.exe`, `OpenClawCompanion-Setup-arm64.exe` en `OpenClawCompanion-SHA256SUMS.txt` op de GitHub-release van OpenClaw. Geef de exacte ondertekende releasetag `openclaw/openclaw-windows-node` door als `windows_node_tag` en de door de kandidaat goedgekeurde digesttoewijzing van het installatieprogramma als `windows_node_installer_digests`; `OpenClaw Release Publish` behoudt het releaseconcept, start `Windows Node Release` en verifieert alle drie de assets vóór publicatie.
12. Voer na publicatie de npm-verificatie na publicatie uit, optioneel de zelfstandige Telegram-E2E voor gepubliceerde npm wanneer bewijs voor het kanaal na publicatie nodig is, promoveer zo nodig de dist-tag, verifieer de gegenereerde GitHub-releasepagina, voer de stappen voor de releaseaankondiging uit en voltooi vervolgens [Afronding van stable main](#stable-main-closeout) voordat je een stable release als voltooid beschouwt.

## Afronding van stable main

Stable publicatie is pas voltooid wanneer `main` de daadwerkelijk uitgebrachte releasestatus bevat.

1. Begin vanaf een verse nieuwste `main`. Controleer `release/YYYY.M.PATCH` hiertegen en port echte correcties die ontbreken in `main` voorwaarts. Voeg niet blindelings uitsluitend voor releases bedoelde compatibiliteits-, test- of validatieadapters samen in de nieuwere `main`.
2. Stel voor het normale pad `main` in op de uitgebrachte stable versie. Bij een late afronding mag `main` worden gebruikt nadat deze is doorgegaan naar een latere stable OpenClaw-CalVer; verlaag een reeds begonnen releasetraject niet uitsluitend om de vorige release af te ronden. De validator vereist nog steeds de exacte uitgebrachte changelogsectie en appcastvermelding en legt de daadwerkelijke versie en SHA van `main` vast. Voer `pnpm release:prep` uit na elke wijziging van de hoofdversie en daarna `pnpm deps:shrinkwrap:generate`.
3. Laat de sectie `## YYYY.M.PATCH` van `CHANGELOG.md` op `main` exact overeenkomen met de getagde releasetakken. Neem de stable update van `appcast.xml` op wanneer de Mac-release er een heeft gepubliceerd.
4. Voeg `YYYY.M.PATCH+1`, een bètaversie of een lege toekomstige changelogsectie pas toe aan `main` wanneer de operator dat releasetraject expliciet start.
5. Voer `pnpm release:generated:check`, `pnpm deps:shrinkwrap:check` en `OPENCLAW_TESTBOX=1 pnpm check:changed` uit. Push en verifieer vervolgens dat `origin/main` de uitgebrachte versie en changelog bevat voordat je de stable release als voltooid beschouwt.
6. Houd de repositoryvariabelen `RELEASE_ROLLBACK_DRILL_ID` en `RELEASE_ROLLBACK_DRILL_DATE` actueel na elke besloten rollbackoefening.

`OpenClaw Stable Main Closeout` begint vanaf de push naar `main` die na stable publicatie de uitgebrachte versie, changelog en appcast bevat. De workflow leest onveranderlijk bewijs na publicatie om de uitgebrachte tag te koppelen aan de bijbehorende runs voor Volledige releasevalidatie en Publicatie, en verifieert vervolgens de status van stable main, de release, de verplichte stable duurtest en blokkerend prestatiebewijs. Er worden een onveranderlijk afrondingsmanifest en controlesom aan de GitHub-release toegevoegd. De automatische push-trigger slaat verouderde releases over die dateren van vóór onveranderlijk bewijs na publicatie en beschouwt die overslag nooit als een voltooide afronding.

Een volledige afronding vereist zowel assets als een overeenkomende controlesom. Een gedeeltelijk manifest speelt de vastgelegde SHA `main` en rollbackoefening opnieuw af om identieke bytes te regenereren en voegt vervolgens de ontbrekende controlesom toe; een ongeldig paar, of een controlesom zonder manifest, blijft blokkerend. Een door een push geactiveerde run zonder repositoryvariabelen voor de rollbackoefening wordt overgeslagen zonder de afronding te voltooien; een ontbrekende of meer dan 90 dagen oude registratie van de oefening blokkeert nog steeds handmatige, door bewijs ondersteunde afronding. Besloten herstelopdrachten blijven in het uitsluitend voor beheerders bestemde draaiboek. Gebruik handmatige activering alleen om een door bewijs ondersteunde stable afronding te herstellen of opnieuw af te spelen.

Als het bovenliggende Release Publish-proces pas mislukte nadat onveranderlijk npm-/pluginbewijs was bijgevoegd, herstel en publiceer dan eerst elke stable platformasset. Daarna mag een beheerder de afronding handmatig starten met `allow_failed_publish_recovery=true`; die modus accepteert uitsluitend een voltooid mislukt bovenliggend proces en vereist bovendien de exacte contracten voor Android- en Windows-assets, GitHub-SHA-256-digests, controlesomverificatie, Android-herkomst en een geslaagde, door het bovenliggende proces gestarte Windows-promotie waarvan de Authenticode-controles en door de kandidaat goedgekeurde digests overeenkomen met de gepubliceerde installatieprogramma's, naast de normale macOS-/appcastcontroles. Automatische afronding via een push schakelt deze herstelmodus nooit in.

Een verouderde correctietag voor terugval mag bewijs van het basispakket alleen hergebruiken wanneer de correctietag naar dezelfde broncommit verwijst als de stable basistag. De Android-release ervan hergebruikt de geverifieerde APK van de basistag en voegt herkomstinformatie voor de correctietag toe. Een correctie met een andere bron moet eigen pakketbewijs publiceren en verifiëren en een hogere Android-`versionCode` gebruiken.

## Releasepreflight

- Voer `pnpm check:test-types` uit vóór de releasepreflight, zodat TypeScript voor tests gedekt blijft buiten de snellere lokale poort `pnpm check`.
- Voer `pnpm check:architecture` uit vóór de releasepreflight, zodat de bredere controles op importcycli en architectuurgrenzen buiten de snellere lokale poort slagen.
- Voer `pnpm build && pnpm ui:build` uit vóór `pnpm release:check`, zodat de verwachte releaseartefacten van `dist/*` en de Control UI-bundel bestaan voor de pakketvalidatiestap.
- Voer `pnpm release:prep` uit na de verhoging van de hoofdversie en vóór het taggen. Hiermee wordt elke deterministische releasegenerator uitgevoerd die vaak afwijkt na een wijziging in versie/configuratie/API: pluginversies, npm-shrinkwraps, plugininventaris, basisschema voor configuratie, configuratiemetadata van gebundelde kanalen, basislijn voor configuratiedocumentatie, exports van de plugin-SDK, het API-contractmanifest van de plugin-SDK en landinstellingsbundels van de Control UI. De stap blokkeert ook totdat vertalingen van native apps en door platforms gegenereerde landinstellingsresources overeenkomen met de broninventaris; als die achterlopen, wacht dan op of start `Native App Locale Refresh` voordat de Code-SHA wordt bevroren. `pnpm release:check` voert die controles opnieuw uit in controlemodus (inclusief de strikte landinstellingspoorten en het oppervlaktebudget van de plugin-SDK) en rapporteert alle fouten door afwijkingen in gegenereerde bestanden in één doorgang voordat de pakketcontroles voor de release worden uitgevoerd.
- Synchronisatie van pluginversies werkt standaard het publiceerbare runtimepakket `@openclaw/ai`, de pakketversies van officiële plugins en bestaande ondergrenzen van `openclaw.compat.pluginApi` bij naar de OpenClaw-releaseversie. Behandel dat veld als de API-ondergrens van de plugin-SDK/runtime, niet slechts als een kopie van de pakketversie: behoud voor uitsluitend pluginreleases die bewust compatibel blijven met oudere OpenClaw-hosts de ondergrens op de oudste ondersteunde host-API en documenteer die keuze in het releasebewijs van de plugin.
- Voer vóór goedkeuring van de release de handmatige workflow `Full Release Validation` uit om alle pre-releasetestboxen vanaf één toegangspunt te starten. De workflow accepteert een branch, tag of volledige commit-SHA, start handmatig `CI` en start `OpenClaw Release Checks` voor installatierooktests, pakketacceptatie, pakketcontroles tussen besturingssystemen, pariteit van QA Lab, Matrix- en Telegram-trajecten. Stable en volledige runs bevatten altijd uitgebreide live-/E2E- en Docker-duurtests voor het releasepad; `run_release_soak=true` blijft beschikbaar voor een expliciete bètaduurtest. Pakketacceptatie levert de canonieke Telegram-E2E voor het pakket tijdens kandidaatvalidatie, waardoor een tweede gelijktijdige live-poller wordt vermeden.

  Geef `release_package_spec` op na publicatie van een bèta om het uitgebrachte npm-pakket opnieuw te gebruiken voor releasecontroles, Pakketacceptatie en Telegram-E2E voor het pakket, zonder de releasetarball opnieuw te bouwen. Geef `npm_telegram_package_spec` alleen op wanneer Telegram een ander gepubliceerd pakket moet gebruiken dan de rest van de releasevalidatie. Geef `package_acceptance_package_spec` op wanneer Pakketacceptatie een ander gepubliceerd pakket moet gebruiken dan de pakketspecificatie van de release. Geef `evidence_package_spec` op wanneer het releasebewijsrapport moet aantonen dat de validatie overeenkomt met een gepubliceerd npm-pakket zonder Telegram-E2E af te dwingen.

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- Voer de handmatige `Package Acceptance`-workflow uit als je aanvullend bewijs via een zijkanaal wilt voor een pakketkandidaat terwijl het releasewerk doorgaat. Gebruik `source=npm` voor `openclaw@beta`, `openclaw@latest` of een exacte releaseversie; `source=ref` om een vertrouwde `package_ref`-branch/tag/SHA te verpakken met de huidige `workflow_ref`-testomgeving; `source=url` voor een openbare HTTPS-tarball met een verplichte SHA-256 en een strikt beleid voor openbare URL's; `source=trusted-url` voor een benoemd beleid voor vertrouwde bronnen met verplichte `trusted_source_id` en SHA-256; of `source=artifact` voor een tarball die door een andere GitHub Actions-run is geüpload.

  De workflow zet de kandidaat om in `package-under-test`, hergebruikt de Docker E2E-releasescheduler voor die tarball en kan Telegram-QA uitvoeren voor dezelfde tarball met `telegram_mode=mock-openai` of `telegram_mode=live-frontier`. Wanneer de geselecteerde Docker-lanes `published-upgrade-survivor` bevatten, is het pakketartefact de kandidaat en selecteert `published_upgrade_survivor_baseline` de gepubliceerde referentieversie. `update-restart-auth` gebruikt het kandidaatpakket zowel als de geïnstalleerde CLI als het te testen pakket, zodat het beheerde herstartpad van de updateopdracht van de kandidaat wordt getest.

  Voorbeeld:

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  Veelgebruikte profielen:
  - `smoke`: lanes voor installatie/kanaal/agent, Gateway-netwerk en het opnieuw laden van configuratie
  - `package`: artefacteigen lanes voor pakket/update/herstart/Plugin zonder OpenWebUI of live ClawHub
  - `product`: pakketprofiel plus MCP-kanalen, opschoning van cron/subagents, OpenAI-zoekopdrachten op het web en OpenWebUI
  - `full`: delen van het Docker-releasepad met OpenWebUI
  - `custom`: exacte selectie van `docker_lanes` voor een gerichte heruitvoering

- Voer de handmatige `CI`-workflow rechtstreeks uit wanneer je alleen deterministische normale CI-dekking voor de releasekandidaat nodig hebt. Handmatige CI-starts omzeilen de gewijzigde scope en forceren de Linux Node-shards, shards voor gebundelde plugins, contractshards voor plugins en kanalen, compatibiliteit met Node 22, `check-*`, `check-additional-*`, smokecontroles voor gebouwde artefacten, documentatiecontroles, Python-Skills, Windows, macOS en de i18n-lanes van de Control UI. Zelfstandige handmatige CI-runs voeren Android alleen uit wanneer ze worden gestart met `include_android=true`; `Full Release Validation` geeft die invoer door aan de onderliggende CI-workflow.

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- Voer `pnpm qa:otel:smoke` uit bij het valideren van releasetelemetrie. Hiermee wordt QA-lab getest via een lokale OTLP/HTTP-ontvanger en worden de export van traces, metrieken en logboeken, begrensde tracekenmerken en het redigeren van inhoud/identificatoren geverifieerd zonder Opik, Langfuse of een andere externe collector te vereisen.
- Voer `pnpm qa:otel:collector-smoke` uit bij het valideren van collectorcompatibiliteit. Hiermee wordt dezelfde OTLP-export van QA-lab via een echte OpenTelemetry Collector-Dockercontainer geleid voordat de controles van de lokale ontvanger worden uitgevoerd.
- Voer `pnpm qa:prometheus:smoke` uit bij het valideren van beveiligde Prometheus-scraping. Hiermee wordt QA-lab getest, worden niet-geverifieerde scrapes geweigerd en wordt gecontroleerd of releasekritieke metriekfamilies vrij blijven van promptinhoud, onbewerkte identificatoren, authenticatietokens en lokale paden.
- Voer `pnpm qa:observability:smoke` uit om de smoke-lanes voor OpenTelemetry en Prometheus vanuit de broncheckout direct na elkaar uit te voeren.
- Voer `pnpm release:check` uit vóór elke getagde release.
- De preflight van `OpenClaw NPM Release` genereert releasebewijs voor afhankelijkheden voordat de npm-tarball wordt verpakt. De kwetsbaarheidspoort voor npm-beveiligingsadviezen blokkeert de release. De rapporten over risico's in het transitieve manifest, eigendom/installatieoppervlak van afhankelijkheden en wijzigingen in afhankelijkheden dienen alleen als releasebewijs. Het rapport over wijzigingen in afhankelijkheden vergelijkt de releasekandidaat met de vorige bereikbare releasetag. De preflight uploadt het afhankelijkheidsbewijs als `openclaw-release-dependency-evidence-<tag>` en neemt het ook op onder `dependency-evidence/` in het voorbereide npm-preflightartefact. Het daadwerkelijke publicatiepad hergebruikt dat preflightartefact en voegt vervolgens hetzelfde bewijs als `openclaw-<version>-dependency-evidence.zip` toe aan de GitHub-release.
- Voer `OpenClaw Release Publish` uit voor de muterende publicatiereeks nadat de tag bestaat. Start reguliere bèta- en stabiele publicaties vanuit de vertrouwde `main`; de releasetag selecteert nog steeds de exacte doelcommit en kan naar `release/YYYY.M.PATCH` verwijzen. Tideclaw-alfapublicaties blijven op hun overeenkomstige alfabranch. Geef de succesvolle OpenClaw npm-`preflight_run_id`, de succesvolle `full_release_validation_run_id` en de exacte `full_release_validation_run_attempt` door en behoud het standaardpublicatiebereik voor plugins `all-publishable`, tenzij je doelbewust een gerichte reparatie uitvoert. De workflow voert de npm-publicatie van plugins, de ClawHub-publicatie van plugins en de npm-publicatie van OpenClaw na elkaar uit, zodat het kernpakket niet vóór de geëxternaliseerde plugins wordt gepubliceerd; de promotie voor Windows en Android wordt gelijktijdig uitgevoerd met de npm-publicatie van de kern voor de conceptreleasepagina. Heruitvoeringen van publicaties kunnen worden hervat: bij een reeds gepubliceerde npm-versie van de kern wordt de kernpublicatie overgeslagen nadat de workflow heeft bewezen dat de tarball in het register overeenkomt met het preflightartefact van de tag, en de promotie voor Windows/Android wordt overgeslagen wanneer de release al het geverifieerde artefactcontract bevat, zodat bij een nieuwe poging alleen de mislukte fasen opnieuw worden uitgevoerd. Voor gerichte reparaties van alleen plugins zijn `plugin_publish_scope=selected` en een niet-lege lijst met plugins vereist. Voor `all-publishable`-runs met alleen plugins is volledig, onveranderlijk bewijs van de preflight en Full Release Validation vereist; gedeeltelijk bewijs wordt geweigerd.
- Voor stabiele `OpenClaw Release Publish` is een exacte `windows_node_tag` vereist nadat de overeenkomstige niet-voorrelease `openclaw/openclaw-windows-node`-release bestaat, plus de door de kandidaat goedgekeurde `windows_node_installer_digests`-toewijzing. Voordat een onderliggende publicatieworkflow wordt gestart, wordt gecontroleerd of die bronrelease gepubliceerd is, geen voorrelease is, de vereiste x64-/ARM64-installatieprogramma's bevat en nog steeds overeenkomt met die goedgekeurde toewijzing. Vervolgens wordt `Windows Node Release` gestart terwijl de OpenClaw-release nog een concept is, waarbij de vastgezette toewijzing van installatieprogrammadigests ongewijzigd wordt doorgegeven. De onderliggende workflow downloadt de ondertekende Windows Hub-installatieprogramma's van die exacte tag, vergelijkt ze met de vastgezette digests, controleert op een Windows-runner of hun Authenticode-handtekeningen de verwachte ondertekenaar van de OpenClaw Foundation gebruiken, schrijft een SHA-256-manifest en uploadt de installatieprogramma's plus het manifest naar de canonieke OpenClaw-release op GitHub. Vervolgens worden de gepromote artefacten opnieuw gedownload en worden het lidmaatschap van het manifest en de hashes geverifieerd. De bovenliggende workflow verifieert vóór publicatie het huidige artefactcontract voor x64, ARM64 en controlesommen. Direct herstel weigert onverwachte `OpenClawCompanion-*`-artefactnamen voordat de verwachte contractartefacten worden vervangen door de vastgezette bronbytes.

  Start `Windows Node Release` alleen handmatig voor herstel en geef altijd een exacte tag door, nooit `latest`, plus de expliciete JSON-toewijzing `expected_installer_digests` uit de goedgekeurde bronrelease. Downloadlinks op de website moeten verwijzen naar exacte URL's van OpenClaw-releaseartefacten voor de huidige stabiele release, of pas naar `releases/latest/download/...` nadat is gecontroleerd dat de omleiding naar de nieuwste release van GitHub naar diezelfde release verwijst; link niet alleen naar de releasepagina van de aanvullende repository.

- Releasecontroles worden nu uitgevoerd in een afzonderlijke handmatige workflow: `OpenClaw Release Checks`. Deze voert vóór releasegoedkeuring ook de mockpariteitslane van QA Lab uit, plus het Matrix-releaseprofiel en de Telegram-QA-lane. De live-lanes gebruiken de `qa-live-shared`-omgeving; Telegram gebruikt daarnaast Convex-CI-leases voor inloggegevens. Voer de handmatige `QA-Lab - All Lanes`-workflow uit met `matrix_profile=all` wanneer je elk onderhouden Matrix-scenario wilt uitvoeren; de workflow verdeelt die selectie over de transport-, media- en E2EE-profielen om het volledige bewijs binnen de time-outs per job te houden.
- Runtimevalidatie van installatie en upgrades op meerdere besturingssystemen maakt deel uit van de openbare `OpenClaw Release Checks` en `Full Release Validation`, die de herbruikbare workflow `.github/workflows/openclaw-cross-os-release-checks-reusable.yml` rechtstreeks aanroepen. Deze opsplitsing is bewust: houd het echte npm-releasepad kort, deterministisch en gericht op artefacten, terwijl tragere live-controles in hun eigen lane blijven zodat ze de publicatie niet vertragen of blokkeren.
- Releasecontroles die geheimen bevatten, moeten worden gestart via `Full Release Validation` of vanaf de `main`/release-workflowref, zodat de workflowlogica en geheimen gecontroleerd blijven.
- `OpenClaw Release Checks` accepteert een branch, tag of volledige commit-SHA zolang de herleide commit bereikbaar is vanuit een OpenClaw-branch of releasetag.
- De preflight van `OpenClaw NPM Release`, die alleen voor validatie dient, accepteert ook de huidige volledige workflowbranch-commit-SHA van 40 tekens zonder dat een gepushte tag vereist is. Dat SHA-pad dient alleen voor validatie en kan niet worden gepromoveerd tot een echte publicatie. In SHA-modus genereert de workflow `v<package.json version>` alleen voor de controle van pakketmetadata; voor een echte publicatie blijft een echte releasetag vereist.
- Beide workflows houden het echte publicatie- en promotiepad op door GitHub gehoste runners, terwijl het niet-muterende validatiepad de grotere Linux-runners van Blacksmith kan gebruiken.
- Die workflow voert `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` uit met zowel de workflowgeheimen `OPENAI_API_KEY` als `ANTHROPIC_API_KEY`.
- De npm-releasepreflight wacht niet langer op de afzonderlijke lane voor releasecontroles.
- Voer `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check` uit voordat je lokaal een release candidate tagt. De helper voert de snelle releasewaarborgen, npm-/ClawHub-releasecontroles voor plugins, de build, de UI-build en `release:openclaw:npm:check` uit in de volgorde die veelvoorkomende fouten die goedkeuring blokkeren detecteert voordat de GitHub-publicatieworkflow start.
- Voer vóór goedkeuring `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts` uit (of de bijbehorende prerelease-/correctietag).
- Voer na de npm-publicatie `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH` uit (of de bijbehorende bèta-/correctieversie) om het installatiepad van het gepubliceerde register in een nieuw tijdelijk voorvoegsel te verifiëren.
- Voer na een bètapublicatie `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live` uit om onboarding van het geïnstalleerde pakket, Telegram-configuratie en echte Telegram-E2E tegen het gepubliceerde npm-pakket te verifiëren met de gedeelde pool van geleasete Telegram-inloggegevens. Voor eenmalige lokale uitvoeringen door maintainers mogen de Convex-variabelen worden weggelaten en kunnen de drie `OPENCLAW_QA_TELEGRAM_*`-omgevingsinloggegevens rechtstreeks worden doorgegeven.
- Gebruik `pnpm release:beta-smoke -- --beta betaN` om de volledige bètasmoketest na publicatie uit te voeren vanaf de machine van een maintainer. De helper voert Parallels-validatie voor npm-updates en nieuwe doelen uit, start `NPM Telegram Beta E2E`, pollt de exacte workflowuitvoering, downloadt het artefact en drukt het Telegram-rapport af.
- Maintainers kunnen dezelfde controle na publicatie vanuit GitHub Actions uitvoeren via de handmatige `NPM Telegram Beta E2E`-workflow. Deze is bewust uitsluitend handmatig en wordt niet bij elke merge uitgevoerd.
- Releaseautomatisering voor maintainers gebruikt eerst preflight en daarna promotie:
  - Een echte npm-publicatie moet een geslaagde npm-`preflight_run_id` doorstaan.
  - De orkestratie en preflight van reguliere bèta- en stabiele publicaties gebruiken vertrouwde `main` tegen de exacte doeltag. De publicatie en preflight van Tideclaw-alpha gebruiken de bijbehorende alphabranch.
  - Stabiele npm-releases gebruiken standaard `beta`; een stabiele npm-publicatie kan via workflowinvoer expliciet op `latest` worden gericht.
  - Tokengebaseerde mutatie van npm-dist-tags bevindt zich in `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`, omdat `npm dist-tag add` nog steeds `NPM_TOKEN` nodig heeft terwijl de bronrepository uitsluitend OIDC-publicatie behoudt.
  - De openbare `macOS Release` dient alleen voor validatie; wanneer een tag uitsluitend op een releasebranch staat maar de workflow vanuit `main` wordt gestart, stel je `public_release_branch=release/YYYY.M.PATCH` in.
  - Een echte macOS-publicatie moet geslaagde macOS-`preflight_run_id` en `validate_run_id` doorstaan.
  - Echte publicatiepaden promoveren voorbereide artefacten in plaats van ze opnieuw te bouwen.
- Voor stabiele correctiereleases zoals `YYYY.M.PATCH-N` controleert de verificatie na publicatie ook hetzelfde upgradepad met tijdelijk voorvoegsel van `YYYY.M.PATCH` naar `YYYY.M.PATCH-N`, zodat releasecorrecties oudere globale installaties niet ongemerkt op de payload van de oorspronkelijke stabiele release laten staan.
- De npm-releasepreflight faalt gesloten tenzij de tarball zowel `dist/control-ui/index.html` als een niet-lege `dist/control-ui/assets/`-payload bevat, zodat we niet opnieuw een leeg browserdashboard uitbrengen.
- De verificatie na publicatie controleert ook of gepubliceerde plugin-entrypoints en pakketmetadata aanwezig zijn in de geïnstalleerde registerindeling. Een release waarin runtimepayloads voor plugins ontbreken, faalt in de postpublish-verificatie en kan niet naar `latest` worden gepromoveerd.
- `pnpm test:install:smoke` handhaaft ook het npm-packbudget `unpackedSize` voor de kandidaat-updatetarball, zodat installer-E2E onbedoelde groei van het pakket detecteert voordat het publicatiepad van de release wordt uitgevoerd.
- Als het releasewerk de CI-planning, timingmanifesten voor extensies of testmatrices voor extensies heeft gewijzigd, genereer en beoordeel dan vóór goedkeuring opnieuw de door de planner beheerde `plugin-prerelease-extension-shard`-matrixuitvoer vanuit `.github/workflows/plugin-prerelease.yml`, zodat de releaseopmerkingen geen verouderde CI-indeling beschrijven.
- Gereedheid voor een stabiele macOS-release omvat ook de updater-oppervlakken: de GitHub-release moet uiteindelijk de verpakte `.zip`, `.dmg` en `.dSYM.zip` bevatten; `appcast.xml` op `main` moet na publicatie naar het nieuwe stabiele zipbestand verwijzen (de macOS-publicatieworkflow commit dit automatisch of opent een appcast-PR wanneer rechtstreeks pushen is geblokkeerd); de verpakte app moet een niet-debug-bundle-id, een niet-lege Sparkle-feed-URL en een `CFBundleVersion` behouden die gelijk is aan of hoger ligt dan de canonieke minimale Sparkle-build voor die releaseversie.

## Testboxen voor releases

`Full Release Validation` is de manier waarop operators de volledige productmatrix vanuit één toegangspunt starten. Gebruik de helper zodat elke onderliggende workflow wordt uitgevoerd vanaf een tijdelijke branch die is vastgezet op één vertrouwde `main`-workflow-SHA, terwijl de aangevraagde commit de te testen kandidaat blijft:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

De helper haalt de huidige `origin/main` op, pusht `release-ci/<workflow-sha>-...` op die vertrouwde workflowcommit, leidt `beta` af uit alpha-/bètapakketversies en anders `stable`, start `Full Release Validation` vanaf de tijdelijke branch met `ref=<target-sha>`, verifieert dat elke `headSha` van een onderliggende workflow overeenkomt met de vastgezette bovenliggende workflow-SHA en verwijdert vervolgens de tijdelijke branch. Geef `-f reuse_evidence=false` door om een nieuwe uitvoering af te dwingen, `-f release_profile=full` voor de brede adviserende sweep of `--workflow-sha <trusted-main-sha>` om een oudere commit vast te zetten die nog steeds bereikbaar is vanuit de huidige `origin/main`. De workflow zelf schrijft nooit repositoryrefs. Hierdoor blijft releasetooling die alleen op main beschikbaar is bruikbaar zonder toolingcommits aan de kandidaat toe te voegen en wordt voorkomen dat per ongeluk bewijs wordt geleverd met een nieuwere onderliggende `main`-uitvoering.

Nadat de Code SHA groen is, commit je alleen `CHANGELOG.md` en voer je dezelfde helper uit met de Release SHA:

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

De tweede bovenliggende workflow hergebruikt productbewijs alleen wanneer GitHub bewijst dat de Release SHA afstamt van de Code SHA en de volledige set gewijzigde paden exact `CHANGELOG.md` is. Deze registreert `changelog-only-release-v1` en start geen onderliggende productworkflows. De npm-preflight en pakket-/installatieacceptatie worden nog steeds op de Release SHA uitgevoerd omdat de bytes van de tarball zijn gewijzigd.

Voor een nieuwe Code SHA herleidt de workflow het doel, start de handmatige `CI` en start vervolgens `OpenClaw Release Checks`. `OpenClaw Release Checks` verdeelt de installatiesmoketest, releasecontroles op meerdere besturingssystemen, live-/E2E-Dockerdekking van het releasepad wanneer soak is ingeschakeld, pakketacceptatie met de canonieke Telegram-pakket-E2E, QA Lab-pariteit, live Matrix en live Telegram. Een volledige/alle-uitvoering is alleen acceptabel wanneer de `Full Release Validation`-samenvatting `normal_ci`, `plugin_prerelease` en `release_checks` als geslaagd toont, tenzij bij een gerichte heruitvoering bewust de afzonderlijke onderliggende `Plugin Prerelease` is overgeslagen. Gebruik de zelfstandige onderliggende `npm-telegram` alleen voor een gerichte heruitvoering van het gepubliceerde pakket met `release_package_spec` of `npm_telegram_package_spec`. De uiteindelijke verificatiesamenvatting bevat tabellen met de traagste jobs voor elke onderliggende uitvoering, zodat de releasemanager het huidige kritieke pad kan zien zonder logs te downloaden.

De onderliggende workflow voor productprestaties gebruikt in dit releasepad uitsluitend artefacten. De
overkoepelende workflow start deze met `publish_reports=false`, en de validatie wordt afgewezen
tenzij de bewaking voor uitsluitend artefacten bewijst dat de publicator van Clawgrit-rapporten
overgeslagen bleef.

Zie [Volledige releasevalidatie](/nl/reference/full-release-validation) voor de volledige fasematrix, exacte namen van workflowjobs, verschillen tussen stabiele en volledige profielen, artefacten en aanknopingspunten voor gerichte heruitvoeringen.

Onderliggende workflows worden gestart vanaf de op SHA vastgezette vertrouwde ref die `Full Release Validation` uitvoert. Elke onderliggende uitvoering moet exact dezelfde bovenliggende workflow-SHA gebruiken. Gebruik geen onbewerkte `--ref main -f ref=<sha>`-starts voor releasebewijs; gebruik `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH`.

Gebruik `release_profile` om de breedte van live-/providerdekking te selecteren:

- `beta`: snelste releasekritieke live- en Dockerpad voor OpenAI/core
- `stable`: bèta plus dekking van stabiele providers/backends voor releasegoedkeuring
- `full`: stabiel plus brede adviserende provider-/mediadekking

Stabiele en volledige validatie voeren vóór promotie altijd de uitputtende live-/E2E-sweep, het Docker-releasepad en de begrensde survivalsweep voor gepubliceerde upgrades uit. Gebruik `run_release_soak=true` om dezelfde sweep voor een bèta aan te vragen. Die sweep dekt de vier nieuwste stabiele pakketten plus vastgezette `2026.4.23`- en `2026.5.2`-baselines en oudere `2026.4.15`-dekking, waarbij dubbele baselines worden verwijderd en elke baseline in een eigen Docker-runnerjob wordt geshard.

`OpenClaw Release Checks` gebruikt de vertrouwde workflowref om de doelref eenmaal als `release-package-under-test` te herleiden en hergebruikt dat artefact in controles voor meerdere besturingssystemen, pakketacceptatie en Docker-controles van het releasepad wanneer soak wordt uitgevoerd. Hierdoor gebruiken alle pakketgerichte boxen dezelfde bytes en worden herhaalde pakketbuilds vermeden. Nadat een bèta al op npm staat, stel je `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` in zodat releasecontroles het uitgebrachte pakket eenmaal downloaden, de bron-SHA van de build uit `dist/build-info.json` extraheren en dat artefact hergebruiken voor meerdere besturingssystemen, pakketacceptatie, releasepad-Docker en Telegram-pakketlanes.

De OpenAI-installatiesmoketest op meerdere besturingssystemen gebruikt `OPENCLAW_CROSS_OS_OPENAI_MODEL` wanneer de repository-/organisatievariabele is ingesteld, en anders `openai/gpt-5.6-luna`, omdat deze lane pakketinstallatie, onboarding, het starten van de Gateway en één live agentbeurt bewijst in plaats van het krachtigste model te benchmarken. De bredere live-providermatrix blijft de plaats voor modelspecifieke dekking.

Gebruik afhankelijk van de releasefase deze varianten:

```bash
# Valideer de productvolledige Code-SHA.
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# Valideer de Release-SHA met alleen changelogwijzigingen door productbewijs van de Code-SHA opnieuw te gebruiken.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# Voeg na het publiceren van een bèta Telegram-E2E voor het gepubliceerde pakket toe.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

Gebruik de volledige overkoepelende workflow niet als eerste nieuwe uitvoering na een gerichte oplossing. Als één box mislukt, gebruik dan de mislukte onderliggende workflow, taak, Docker-lane, het pakketprofiel, de modelprovider of de QA-lane voor het volgende bewijs. Voer de volledige overkoepelende workflow alleen opnieuw uit wanneer de oplossing de gedeelde release-orkestratie heeft gewijzigd of eerder bewijs voor alle boxen verouderd heeft gemaakt. De uiteindelijke verificatie van de overkoepelende workflow controleert de vastgelegde uitvoerings-id's van onderliggende workflows opnieuw. Voer daarom, nadat een onderliggende workflow opnieuw met succes is uitgevoerd, alleen de mislukte bovenliggende taak `Verify full validation` opnieuw uit.

`rerun_group=all` kan een eerdere geslaagde uitvoering van de overkoepelende workflow hergebruiken wanneer het releaseprofiel,
de effectieve soak-instelling en de validatie-invoer overeenkomen en de doel-SHA
identiek is, of het nieuwe doel een afstammeling is waarvan de volledige verzameling gewijzigde paden
exact `CHANGELOG.md` is. Hergebruik van het exacte doel legt
`exact-target-full-validation-v1` vast; de Release-SHA na validatie legt
`changelog-only-release-v1` vast. Die laatste hergebruikt alleen productvalidatie. Npm-
preflight, pakketbytes, herkomst van releaseopmerkingen en acceptatie van installatie/updates
moeten nog steeds tegen de Release-SHA worden uitgevoerd. Elke wijziging aan een doel dat eigendom is van een versie, bron, gegenereerde
uitvoer, afhankelijkheid, pakket of workflow vereist een nieuwe Code-SHA
en een nieuwe volledige validatie. Nieuwere uitvoeringen van de overkoepelende workflow voor dezelfde `release/*`-ref en
groep voor nieuwe uitvoeringen vervangen automatisch nog lopende uitvoeringen. Geef
`reuse_evidence=false` door om een nieuwe volledige uitvoering af te dwingen.

Geef voor begrensd herstel `rerun_group` door aan de overkoepelende workflow. `all` is de werkelijke uitvoering voor de releasekandidaat, `ci` voert alleen de normale onderliggende CI-workflow uit, `plugin-prerelease` voert alleen de onderliggende workflow voor uitsluitend releaseplugins uit, `release-checks` voert elke releasebox uit en de beperktere releasegroepen zijn `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` en `npm-telegram`. Gerichte nieuwe uitvoeringen van `npm-telegram` vereisen `release_package_spec` of `npm_telegram_package_spec`; volledige/alle uitvoeringen gebruiken de canonieke Telegram-E2E voor pakketten binnen Pakketacceptatie. Aan gerichte nieuwe uitvoeringen voor meerdere besturingssystemen kan `cross_os_suite_filter=windows/packaged-upgrade` of een ander filter voor besturingssysteem/suite worden toegevoegd. Mislukte QA-releasecontroles blokkeren de normale releasevalidatie, waaronder afwijkingen van dynamische OpenClaw-tools in de lane voor runtimeparen van de core. Tideclaw-alfa-uitvoeringen mogen releasecontrolelanes die niet over pakketveiligheid gaan nog steeds als adviserend behandelen. Met `release_profile=beta` zijn de liveprovidersuites van `Run repo/live E2E validation` adviserend (waarschuwingen, geen blokkades); in stabiele en volledige profielen blijven ze blokkerend. Wanneer `live_suite_filter` expliciet om een afgeschermde live QA-lane zoals Discord, WhatsApp of Slack vraagt, moet de overeenkomende repositoryvariabele `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` zijn ingeschakeld; anders mislukt het vastleggen van invoer in plaats van de lane stilzwijgend over te slaan.

### Vitest

De Vitest-box is de handmatige onderliggende workflow `CI`. Handmatige CI omzeilt opzettelijk de afbakening op basis van wijzigingen en dwingt de normale testgrafiek voor de releasekandidaat af: Linux Node-shards, shards voor gebundelde plugins, contractshards voor plugins en kanalen, compatibiliteit met Node 22, `check-*`, `check-additional-*`, smokecontroles voor gebouwde artefacten, documentatiecontroles, Python-skills, Windows, macOS en Control UI-i18n. Android wordt opgenomen wanneer `Full Release Validation` de box uitvoert, omdat de overkoepelende workflow `include_android=true` doorgeeft; zelfstandige handmatige CI vereist `include_android=true` voor Android-dekking.

Gebruik deze box om de vraag te beantwoorden: "heeft de bronstructuur de volledige normale testsuite doorstaan?" Dit is niet hetzelfde als productvalidatie van het releasepad. Te bewaren bewijs:

- `Full Release Validation`-samenvatting met de URL van de gestarte `CI`-uitvoering
- geslaagde `CI`-uitvoering op de exacte doel-SHA
- namen van mislukte of trage shards uit de CI-taken bij onderzoek naar regressies
- Vitest-tijdartefacten zoals `.artifacts/vitest-shard-timings.json` wanneer de prestaties van een uitvoering moeten worden geanalyseerd

Voer handmatige CI alleen rechtstreeks uit wanneer de release deterministische normale CI nodig heeft, maar niet de Docker-, QA Lab-, live-, platformoverschrijdende of pakketboxen. Gebruik de eerste opdracht voor rechtstreekse CI zonder Android. Voeg `include_android=true` toe wanneer rechtstreekse CI voor een releasekandidaat Android moet omvatten:

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

De Docker-box bevindt zich in `OpenClaw Release Checks` tot en met `openclaw-live-and-e2e-checks-reusable.yml`, plus de workflow `install-smoke` in releasemodus. Deze valideert de releasekandidaat via verpakte Docker-omgevingen in plaats van uitsluitend tests op bronniveau.

Docker-dekking voor releases omvat:

- volledige installatiesmoke met de trage smoke voor globale Bun-installatie ingeschakeld
- voorbereiding/hergebruik van de smoke-image van het Dockerfile in de hoofdmap per doel-SHA, waarbij QR-, root/Gateway- en installatieprogramma/Bun-smoketaken als afzonderlijke installatiesmokeshards worden uitgevoerd
- E2E-lanes van de repository
- Docker-segmenten voor het releasepad: `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` tot en met `plugins-runtime-install-h` en `openwebui`
- OpenWebUI-dekking op een speciale runner met een grote schijf wanneer daarom wordt gevraagd
- gesplitste lanes voor installatie/verwijdering van gebundelde plugins, `bundled-plugin-install-uninstall-0` tot en met `bundled-plugin-install-uninstall-23`
- live-/E2E-providersuites en dekking van live Docker-modellen wanneer releasecontroles livesuites omvatten

Gebruik Docker-artefacten voordat je opnieuw uitvoert. De planner voor het releasepad uploadt `.artifacts/docker-tests/` met lanelogboeken, `summary.json`, `failures.json`, fasetijden, JSON van het plannerplan en opdrachten voor nieuwe uitvoeringen. Gebruik voor gericht herstel `docker_lanes=<lane[,lane]>` in de herbruikbare live-/E2E-workflow in plaats van alle releasesegmenten opnieuw uit te voeren. Gegenereerde opdrachten voor nieuwe uitvoeringen bevatten waar beschikbaar eerdere `package_artifact_run_id`-invoer en voorbereide Docker-image-invoer, zodat een mislukte lane dezelfde tarball en GHCR-images kan hergebruiken.

### QA Lab

De QA Lab-box maakt ook deel uit van `OpenClaw Release Checks`. Dit is de releasegate voor agentgedrag en gedrag op kanaalniveau, los van Vitest en de pakketmechanismen van Docker.

QA Lab-dekking voor releases omvat:

- mockpariteitslane die de OpenAI-kandidaat-lane met de `anthropic/claude-opus-4-8`-basislijn vergelijkt met behulp van het agentpariteitspakket
- Matrix-releaseprofiel voor live-adapters dat de `qa-live-shared`-omgeving gebruikt
- live Telegram-QA-lane die Convex-CI-leases voor inloggegevens gebruikt
- `pnpm qa:otel:smoke`, `pnpm qa:otel:collector-smoke`, `pnpm qa:prometheus:smoke` of `pnpm qa:observability:smoke` wanneer releasetelemetrie expliciet lokaal bewijs nodig heeft

Gebruik deze box om de vraag te beantwoorden: "gedraagt de release zich correct in QA-scenario's en live kanaalflows?" Bewaar bij het goedkeuren van de release de artefact-URL's voor de pariteits-, Matrix- en Telegram-lanes. Volledige Matrix-dekking blijft beschikbaar als een handmatige gesharde QA Lab-uitvoering in plaats van als de standaard releasekritieke lane.

### Pakket

De pakketbox is de gate voor het installeerbare product. Deze wordt ondersteund door `Package Acceptance` en de resolver `scripts/resolve-openclaw-package-candidate.mjs`. De resolver normaliseert een kandidaat tot de `package-under-test`-tarball die door Docker-E2E wordt gebruikt, valideert de pakketinhoud, legt de pakketversie en SHA-256 vast en houdt de ref van het workflowharnas gescheiden van de ref van de pakketbron.

Ondersteunde kandidaatbronnen:

- `source=npm`: `openclaw@beta`, `openclaw@latest` of een exacte OpenClaw-releaseversie
- `source=ref`: verpak een vertrouwde `package_ref`-branch, -tag of volledige commit-SHA met het geselecteerde `workflow_ref`-harnas
- `source=url`: download een openbare HTTPS-`.tgz` met vereiste `package_sha256`; URL-inloggegevens, niet-standaard HTTPS-poorten, private/interne/voor speciaal gebruik bestemde hostnamen of omgezette adressen en onveilige omleidingen worden geweigerd
- `source=trusted-url`: download een HTTPS-`.tgz` met vereiste `package_sha256` en `trusted_source_id` uit een benoemd beleid in `.github/package-trusted-sources.json`; gebruik dit voor zakelijke mirrors of private pakketrepository's die door beheerders worden beheerd, in plaats van een omzeiling voor private netwerken op invoerniveau toe te voegen aan `source=url`
- `source=artifact`: hergebruik een `.tgz` die door een andere GitHub Actions-uitvoering is geüpload

`OpenClaw Release Checks` voert Pakketacceptatie uit met `source=artifact`, het voorbereide releasepakketartefact, `suite_profile=custom`, `docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`, `telegram_mode=mock-openai`. Pakketacceptatie behoudt migratie, update, door root beheerde VPS-upgrade, herstart na update met geconfigureerde authenticatie, live-installatie van een ClawHub-skill, opschoning van verouderde plugin-afhankelijkheden, offline pluginfixtures, plugin-update, beveiliging tegen escaping van pluginopdrachtbindingen en Telegram-pakket-QA tegen dezelfde omgezette tarball. Blokkerende releasecontroles gebruiken standaard het nieuwste gepubliceerde pakket als basislijn; het bètaprofiel met `run_release_soak=true`, `release_profile=stable` of `release_profile=full` breidt de controle van overlevende gepubliceerde upgrades uit naar `last-stable-4` plus de vastgezette basislijnen `2026.4.23`, `2026.5.2` en `2026.4.15` met `reported-issues`-scenario's. Gebruik Pakketacceptatie met `source=npm` voor een reeds uitgebrachte kandidaat, `source=ref` voor een door een SHA ondersteunde lokale npm-tarball vóór publicatie, `source=trusted-url` voor een zakelijke/private mirror die door een beheerder wordt beheerd, of `source=artifact` voor een voorbereide tarball die door een andere GitHub Actions-uitvoering is geüpload.

Dit is de GitHub-native vervanging voor het grootste deel van de pakket-/updatedekking waarvoor eerder Parallels nodig was. Platformoverschrijdende releasecontroles blijven belangrijk voor platformspecifieke onboarding, installatieprogramma's en platformgedrag, maar voor productvalidatie van pakketten/updates moet de voorkeur uitgaan naar Pakketacceptatie.

De canonieke checklist voor validatie van updates en plugins is [Updates en plugins testen](/nl/help/testing-updates-plugins). Gebruik deze om te bepalen welke lokale, Docker-, Pakketacceptatie- of releasecontrolelane een wijziging aan plugininstallatie/-update, doctor-opschoning of migratie van een gepubliceerd pakket bewijst. Uitputtende migratie van gepubliceerde updates vanaf elk stabiel `2026.4.23+`-pakket is een afzonderlijke handmatige `Update Migration`-workflow en maakt geen deel uit van de volledige release-CI.

De tolerantie voor verouderde pakketacceptatie is opzettelijk in de tijd begrensd. Pakketten tot en met `2026.4.25` mogen het compatibiliteitspad gebruiken voor hiaten in metadata die al naar npm zijn gepubliceerd: private QA-inventarisitems die ontbreken in de tarball, ontbrekende `gateway install --wrapper`, ontbrekende patchbestanden in de van de tarball afgeleide gitfixture, ontbrekende persistente `update.channel`, verouderde locaties van plugininstallatierecords, ontbrekende persistentie van marketplace-installatierecords en migratie van configuratiemetadata tijdens `plugins update`. Het gepubliceerde `2026.4.26`-pakket mag waarschuwen voor stempelbestanden met lokale buildmetadata die al zijn uitgebracht. Latere pakketten moeten aan de moderne pakketcontracten voldoen; diezelfde hiaten doen de releasevalidatie mislukken.

Gebruik bredere Pakketacceptatieprofielen wanneer de releasevraag over een daadwerkelijk installeerbaar pakket gaat:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

Veelgebruikte pakketprofielen:

- `smoke`: snelle banen voor pakketinstallatie/kanaal/agent, Gateway-netwerk en herladen van configuratie
- `package`: contracten voor installatie/update/herstart/Plugin-pakketten plus live bewijs van installatie van ClawHub-Skills; dit is de standaard voor releasecontroles
- `product`: `package` plus MCP-kanalen, opschoning van Cron/subagents, OpenAI-webzoekopdrachten en OpenWebUI
- `full`: Docker-segmenten voor het releasepad met OpenWebUI
- `custom`: exacte lijst met `docker_lanes` voor gerichte herhalingen

Schakel voor Telegram-bewijs met een pakketkandidaat `telegram_mode=mock-openai` of `telegram_mode=live-frontier` in bij Package Acceptance. De workflow geeft het omgezette `package-under-test`-tarball door aan de Telegram-baan; de zelfstandige Telegram-workflow accepteert nog steeds een gepubliceerde npm-specificatie voor controles na publicatie.

## Automatisering voor reguliere releasepublicatie

Voor bèta, `latest`, Plugin, GitHub Release en platformpublicatie
is `OpenClaw Release Publish` het normale muterende toegangspunt. Het maandelijkse
uitgebreid stabiele Gateway-pad `.33+` gebruikt deze orchestrator niet. De
reguliere workflow orkestreert de workflows voor vertrouwde uitgevers in de volgorde die de
release vereist:

1. Check de releasetag uit en bepaal de commit-SHA ervan.
2. Controleer of de tag bereikbaar is vanuit `main` of `release/*` (of een Tideclaw-alfabranch voor alfapre-releases).
3. Voer `pnpm plugins:sync:check` uit.
4. Start `Plugin NPM Release` met `publish_scope=all-publishable` en `ref=<release-sha>`.
5. Start `Plugin ClawHub Release` met hetzelfde bereik en dezelfde SHA.
6. Start `OpenClaw NPM Release` met de releasetag, npm-dist-tag en opgeslagen `preflight_run_id`, nadat de opgeslagen `full_release_validation_run_id` en de exacte uitvoeringspoging zijn gecontroleerd.
7. Maak voor stabiele releases de GitHub-release als concept aan of werk deze bij, start `Windows Node Release` met de expliciete `windows_node_tag` en door de kandidaat goedgekeurde `windows_node_installer_digests`, en controleer de canonieke Windows-installatieprogramma-/checksum-assets. Start ook `Android Release` om de ondertekende APK met exacte tag plus checksum en herkomst te bouwen. Controleer beide contracten voor systeemeigen assets voordat je het concept publiceert.

Voorbeeld van bètapublicatie:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Stabiele publicatie naar de standaard bèta-dist-tag:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Stabiele promotie rechtstreeks naar `latest` is expliciet:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

Gebruik de workflows op lager niveau `Plugin NPM Release` en `Plugin ClawHub Release` alleen voor gericht herstel- of herpublicatiewerk. `OpenClaw Release Publish` weigert `plugin_publish_scope=selected` wanneer `publish_openclaw_npm=true`, zodat het kernpakket niet kan worden uitgebracht zonder elke publiceerbare officiële Plugin, waaronder `@openclaw/diffs-language-pack`. Stel voor herstel van een geselecteerde Plugin `publish_openclaw_npm=false` in met `plugin_publish_scope=selected` en `plugins=@openclaw/name`, of start de onderliggende workflow rechtstreeks.

De initiële ClawHub-bootstrap vormt de uitzondering: start `Plugin ClawHub New`
vanuit de vertrouwde `main` en geef de volledige SHA van de doelrelease door via `ref`.
Voer de bootstrapworkflow zelf nooit uit vanuit de releasetag of -branch:

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

Validatie vóór tagging vereist `dry_run=true`, weigert invoer voor releasetags en bovenliggende uitvoeringen
en accepteert alleen een exact doel dat bereikbaar is vanuit `main` of `release/*`.
Hierbij worden geen ClawHub-inloggegevens geladen, pakketbytes gepubliceerd of configuraties van vertrouwde
uitgevers gewijzigd. De workflow bepaalt nog steeds het actuele registerplan,
checkt het doel uit en verpakt het uitsluitend in een job zonder geheimen, materialiseert de
vergrendelde ClawHub-toolchain en valideert het onveranderlijke artefact en de
pakket-slug/-identiteit voordat de releasetag bestaat. Keur de
omgeving `clawhub-plugin-bootstrap` pas goed nadat de verpakkingsjobs zonder geheimen
zijn voltooid; deze beveiligde validatiejob bevat geen inloggegevens of mutatiecommando's.

Een goedgekeurde proefuitvoering of echte bootstrap na tagging moet de exacte
releasetag plus de uitvoerings-id, poging en
branch van de bovenliggende `OpenClaw Release Publish` bevatten. De bovenliggende uitvoering attesteert de eigen workflow-SHA en een afzonderlijke exacte vertrouwde
SHA `main` voor `Plugin ClawHub New`; de onderliggende uitvoering en elke goedkeuring van een beveiligde
omgeving moeten overeenkomen met die goedgekeurde onderliggende SHA. De releasetag wordt
vóór elke publicatiepoging en mutatie door een vertrouwde uitgever opnieuw gecontroleerd.

De verpakkingsjob
uploadt één onveranderlijk artefact waarvan de naam, Actions-artefact-id/-digest,
producerende uitvoering/poging, doel-SHA en SHA-256/grootte per pakkettarball
worden doorgegeven aan de validatie- en beveiligde jobs. De beveiligde job checkt uitsluitend vertrouwde
`main`-tooling uit, valideert de artefacttuple via de GitHub-API, downloadt
op exacte artefact-id, berekent de hash van elk tarball opnieuw en valideert lokale TAR-paden en
pakketidentiteit met de USTAR-canonicalisatieregels van de vastgezette CLI. Elke
kandidaat doorloopt vervolgens de proefpublicatie van de vastgezette CLI, die terugkeert vóór
registeropzoeking of authenticatie. Het voorfilter van de inloggegevensjob beperkt gecomprimeerde ClawPacks
tot 120 MiB, de totale bestandsinhoud tot 50 MiB, uitgepakte TAR-gegevens tot 64 MiB en
het aantal TAR-items tot 10,000. Herstel van een vertrouwde uitgever voor een bestaand pakket blijft
uitsluitend configureren, maar verpakt nog steeds het doel en vereist de aangevraagde tag
plus exacte gelijkheid van registerbytes en metadata voordat de configuratie van de vertrouwde uitgever
wordt gewijzigd. Verificatie na publicatie downloadt het ClawHub-artefact en
vereist dezelfde SHA-256 en grootte. Bij herstel door opnieuw uitvoeren van mislukte jobs mag het pakketartefact van een eerdere
poging alleen worden hergebruikt wanneer de exacte producerende job
met succes is voltooid. Het uiteindelijke bewijs bindt ook de vergrendelde ClawHub-versie, de
SHA-256 van de vergrendeling en de npm-integriteit. Bij een afwijking is een nieuwe pakketversie vereist.

## Invoer voor de NPM-workflow

`OpenClaw NPM Release` accepteert deze door de operator beheerde invoer:

- `tag`: vereiste releasetag, zoals `v2026.4.2`, `v2026.4.2-1`, `v2026.4.2-beta.1` of `v2026.4.2-alpha.1`; wanneer `preflight_only=true`, mag dit ook de huidige volledige commit-SHA van 40 tekens van de workflowbranch zijn voor uitsluitend een validatievoorcontrole
- `preflight_only`: `true` voor uitsluitend validatie/build/pakket, `false` voor het echte publicatiepad
- `preflight_run_id`: id van een bestaande geslaagde voorcontrole-uitvoering, vereist op het echte publicatiepad zodat de workflow het voorbereide tarball hergebruikt in plaats van het opnieuw te bouwen
- `full_release_validation_run_id`: id van een geslaagde `Full Release Validation`-uitvoering voor deze tag/SHA, vereist voor echte publicatie. Bètapublicaties mogen met een waarschuwing doorgaan op basis van alleen de voorcontrole, maar voor promotie naar stable/`latest` blijft dit vereist.
- `full_release_validation_run_attempt`: exacte positieve uitvoeringspoging gekoppeld aan `full_release_validation_run_id`; vereist wanneer de uitvoerings-id wordt opgegeven, zodat heruitvoeringen het autorisatiebewijs tijdens publicatie niet kunnen wijzigen.
- `release_publish_run_id`: id van de goedgekeurde `OpenClaw Release Publish`-uitvoering; vereist wanneer deze workflow door die bovenliggende workflow wordt gestart (aanroepen voor echte publicatie door een botactor)
- `plugin_npm_run_id`: id van een geslaagde exact-head-uitvoering van `Plugin NPM Release`; vereist voor een echte publicatie van de `extended-stable`-kern
- `npm_dist_tag`: npm-doeltag voor het publicatiepad; accepteert `alpha`, `beta`, `latest` of `extended-stable` en gebruikt standaard `beta`. De definitieve patch `33` en later moeten `extended-stable` gebruiken; standaard weigert `extended-stable` eerdere patches en niet-definitieve tags worden altijd geweigerd.
- `bypass_extended_stable_guard`: booleaanse waarde uitsluitend voor tests, standaard `false`; met `npm_dist_tag=extended-stable` wordt de maandelijkse geschiktheidscontrole voor uitgebreid stabiel omzeild, terwijl controles van release-identiteit, artefact, goedkeuring en teruglezing behouden blijven.

`Plugin NPM Release` accepteert `npm_dist_tag=default` voor bestaand releasegedrag
of `npm_dist_tag=extended-stable` voor het beveiligde maandelijkse pad. De
uitgebreid stabiele optie vereist `publish_scope=all-publishable`, een lege
`plugins`-invoer, een definitieve patch op of boven `33` en de canonieke
`extended-stable/YYYY.M.33`-branch op de exacte tip. Deze verplaatst nooit Plugin-
`latest` of `beta`. Nieuwe pakketversies krijgen `extended-stable` atomair
via vertrouwde OIDC-publicatie (`npm publish --tag extended-stable`); deze
bronworkflow gebruikt geen met een token geauthenticeerde `npm dist-tag add`. Nieuwe pogingen
slaan exacte versies over die al in npm aanwezig zijn en stoppen vervolgens gesloten, tenzij volledige
teruglezing bevestigt dat elk exact pakket en elke `extended-stable`-tag is geconvergeerd.

`OpenClaw Release Publish` accepteert deze door de operator beheerde invoer:

- `tag`: vereiste releasetag; moet al bestaan
- `preflight_run_id`: id van een geslaagde `OpenClaw NPM Release`-voorcontrole-uitvoering; vereist wanneer `publish_openclaw_npm=true` of `plugin_publish_scope=all-publishable`
- `full_release_validation_run_id`: id van een geslaagde `Full Release Validation`-uitvoering; vereist wanneer `publish_openclaw_npm=true` of `plugin_publish_scope=all-publishable`
- `full_release_validation_run_attempt`: exacte positieve poging gekoppeld aan `full_release_validation_run_id`; vereist wanneer de uitvoerings-id wordt opgegeven
- `windows_node_tag`: exacte niet-pre-release `openclaw/openclaw-windows-node`-releasetag; vereist voor stabiele OpenClaw-publicatie
- `windows_node_installer_digests`: door de kandidaat goedgekeurde compacte JSON-toewijzing van de huidige namen van Windows-installatieprogramma's aan hun vastgezette `sha256:`-digests; vereist voor stabiele OpenClaw-publicatie
- `npm_telegram_run_id`: optionele id van een geslaagde `NPM Telegram Beta E2E`-uitvoering om in het uiteindelijke releasebewijs op te nemen
- `npm_dist_tag`: npm-doeltag voor het OpenClaw-pakket, een van `alpha`, `beta` of `latest`
- `plugin_publish_scope`: standaard `all-publishable`; gebruik `selected` alleen voor gericht herstelwerk uitsluitend voor Plugins met `publish_openclaw_npm=false`
- `plugins`: door komma's gescheiden `@openclaw/*`-pakketnamen wanneer `plugin_publish_scope=selected`
- `publish_openclaw_npm`: standaard `true`; stel `false` alleen in wanneer de workflow als orchestrator voor uitsluitend Plugin-herstel wordt gebruikt
- `release_profile`: releasedekkingsprofiel dat wordt gebruikt voor samenvattingen van releasebewijs; standaard `from-validation`, waarmee het uit het validatiemanifest wordt gelezen, of overschrijf dit met `beta`, `stable` of `full`
- `wait_for_clawhub`: standaard `false`, zodat npm-beschikbaarheid niet wordt geblokkeerd door de ClawHub-sidecar; stel `true` alleen in wanneer de voltooiing van de workflow ook de voltooiing van ClawHub moet omvatten

`OpenClaw Release Checks` accepteert deze door de operator beheerde invoer:

- `ref`: branch, tag of volledige commit-SHA om te valideren. Voor controles met geheimen moet de herleide commit bereikbaar zijn vanuit een OpenClaw-branch of releasetag.
- `run_release_soak`: schakel uitgebreide live-/E2E-controles, het Docker-releasepad en langdurige overlevingstests voor upgrades sinds alle eerdere versies in voor bètareleasecontroles. Dit wordt verplicht ingeschakeld door `release_profile=stable` en `release_profile=full`.

Regels:

- Reguliere definitieve en correctieversies onder patch `33` mogen naar `beta` of `latest` worden gepubliceerd. Definitieve versies met patch `33` of hoger moeten naar `extended-stable` worden gepubliceerd en versies met een correctieachtervoegsel op die grens worden geweigerd.
- Bèta-prereleasetags mogen alleen naar `beta` worden gepubliceerd; alfa-prereleasetags mogen alleen naar `alpha` worden gepubliceerd
- Voor `OpenClaw NPM Release` is invoer van een volledige commit-SHA alleen toegestaan wanneer `preflight_only=true`
- `OpenClaw Release Checks` en `Full Release Validation` zijn altijd uitsluitend voor validatie
- Het echte publicatiepad moet dezelfde `npm_dist_tag` gebruiken als tijdens de preflight; de workflow verifieert die metadata voordat de publicatie wordt voortgezet

## Reguliere releasevolgorde voor bèta/nieuwste stabiele versie

Deze verouderde volgorde is bedoeld voor de reguliere georkestreerde release die ook Plugins, GitHub Release, Windows en ander platformwerk omvat. Dit is niet het maandelijkse uitgebreide stabiele Gateway-pad voor `.33+` dat bovenaan deze pagina wordt beschreven.

Bij het uitbrengen van een reguliere georkestreerde stabiele release:

1. Voer `OpenClaw NPM Release` uit met `preflight_only=true`. Voordat er een tag bestaat, kun je de huidige volledige commit-SHA van de workflowbranch gebruiken voor een uitsluitend validerende proefuitvoering van de preflightworkflow.
2. Kies `npm_dist_tag=beta` voor de normale workflow die met een bèta begint, of alleen `latest` wanneer je bewust rechtstreeks een stabiele versie wilt publiceren.
3. Voer `Full Release Validation` uit op de releasebranch, releasetag of volledige commit-SHA wanneer je normale CI plus dekking voor de live promptcache, Docker, QA Lab, Matrix en Telegram vanuit één handmatige workflow wilt. Als je bewust alleen de deterministische normale testgraaf nodig hebt, voer dan in plaats daarvan de handmatige `CI`-workflow uit op de releasereferentie.
4. Selecteer exact de niet-prerelease `openclaw/openclaw-windows-node`-releasetag waarvan de ondertekende x64- en ARM64-installatieprogramma's moeten worden uitgebracht. Sla deze op als `windows_node_tag` en sla de gevalideerde digest-toewijzing ervan op als `windows_node_installer_digests`. De releasekandidaathulp registreert beide en neemt ze op in de gegenereerde publicatieopdracht.
5. Sla de geslaagde `preflight_run_id`, `full_release_validation_run_id` en exacte `full_release_validation_run_attempt` op.
6. Voer `OpenClaw Release Publish` uit vanuit de vertrouwde `main` met dezelfde `tag`, dezelfde `npm_dist_tag`, de geselecteerde `windows_node_tag`, de opgeslagen `windows_node_installer_digests` ervan, de opgeslagen `preflight_run_id`, `full_release_validation_run_id` en `full_release_validation_run_attempt`. Hiermee worden geëxternaliseerde Plugins naar npm en ClawHub gepubliceerd voordat het OpenClaw-npm-pakket wordt gepromoveerd.
7. Als de release op `beta` is terechtgekomen, gebruik je de `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`-workflow om die stabiele versie van `beta` naar `latest` te promoveren.
8. Als de release bewust rechtstreeks naar `latest` is gepubliceerd en `beta` onmiddellijk dezelfde stabiele build moet volgen, gebruik je diezelfde releaseworkflow om beide dist-tags naar de stabiele versie te laten verwijzen, of laat je de geplande zelfherstellende synchronisatie `beta` later verplaatsen.

De wijziging van de dist-tag bevindt zich in de releaseledger-repository omdat hiervoor nog steeds `NPM_TOKEN` vereist is, terwijl de bronrepository uitsluitend via OIDC publiceert. Zo blijven zowel het rechtstreekse publicatiepad als het promotiepad dat met een bèta begint gedocumenteerd en zichtbaar voor operators.

Als een beheerder moet terugvallen op lokale npm-authenticatie, voer dan alle opdrachten van de 1Password-CLI (`op`) uitsluitend uit binnen een afzonderlijke tmux-sessie. Roep `op` niet rechtstreeks aan vanuit de hoofdshell van de agent; door dit binnen tmux te houden, blijven prompts, waarschuwingen en OTP-afhandeling waarneembaar en worden herhaalde waarschuwingen op de host voorkomen.

## Openbare referenties

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Beheerders gebruiken de privé-releasedocumentatie in [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) voor het daadwerkelijke draaiboek.

## Gerelateerd

- [Releasekanalen](/nl/install/development-channels)
