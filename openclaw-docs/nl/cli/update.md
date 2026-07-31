---
read_when:
    - Je wilt een broncode-checkout veilig bijwerken
    - Je debugt de uitvoer of opties van `openclaw update`
    - Je moet het gedrag van de verkorte notatie `--update` begrijpen
summary: CLI-referentie voor `openclaw update` (redelijk veilige bronupdate + automatische herstart van de Gateway)
title: Bijwerken
x-i18n:
    generated_at: "2026-07-27T05:29:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

Werk OpenClaw bij en wissel tussen de kanalen stable/extended-stable/beta/dev.

Als je via **npm/pnpm/bun** hebt geïnstalleerd (globale installatie, zonder git-metadata),
verlopen updates via de pakketbeheerflow die wordt beschreven in
[Bijwerken](/nl/install/updating).

## Gebruik

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` wordt herschreven naar `openclaw update` (handig voor shells en
startscripts).

## Opties

| Vlag                                             | Beschrijving                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | Sla het opnieuw starten van de Gateway-service over na een geslaagde update. Updates via pakketbeheer die wel opnieuw starten, controleren of de opnieuw gestarte service de verwachte versie rapporteert voordat de opdracht slaagt.                                                                                                         |
| `--channel <stable\|extended-stable\|beta\|dev>` | Stel het updatekanaal in en bewaar dit nadat de kernupdate is geslaagd. Extended-stable is alleen voor pakketten beschikbaar.                                                                                                                                                                                                                  |
| `--tag <dist-tag\|version\|spec>`                | Overschrijf het pakketdoel alleen voor deze update. Dit kan niet worden gecombineerd met een actief `extended-stable`-kanaal, waarvoor het geverifieerde exacte doel verplicht is. Voor andere pakketinstallaties wordt `main` gekoppeld aan `github:openclaw/openclaw#main`; GitHub/git-bronspecificaties worden vóór de gefaseerde globale npm-installatie in een tijdelijk tarball verpakt. |
| `--dry-run`                                      | Bekijk een voorbeeld van de geplande acties (kanaal/tag/doel/herstartflow) zonder configuratie te schrijven, te installeren, plugins te synchroniseren of opnieuw te starten.                                                                                                                                                                 |
| `--json`                                         | Druk machineleesbare `UpdateRunResult`-JSON af. Bevat `postUpdate.plugins.warnings` wanneer een beheerde plugin reparatie nodig heeft, details over de fallback voor plugins in het bètakanaal en `postUpdate.plugins.integrityDrifts` wanneer tijdens de synchronisatie na de update afwijkingen in npm-pluginartefacten worden gedetecteerd. |
| `--timeout <seconds>`                            | Time-out per stap. Standaard `1800`.                                                                                                                                                                                                                                                                                              |
| `--yes`                                          | Sla bevestigingsvragen over (bijvoorbeeld de bevestiging van een downgrade).                                                                                                                                                                                                                                                                  |
| `--acknowledge-clawhub-risk`                     | Sta toe dat plugins na de update verder worden gesynchroniseerd ondanks ClawHub-vertrouwenswaarschuwingen voor communitypakketten, zonder interactieve vraag. Zonder deze optie worden riskante communityreleases overgeslagen en ongewijzigd gelaten wanneer OpenClaw geen vraag kan stellen. Officiële ClawHub-pakketten en gebundelde pluginbronnen slaan deze vraag over. |

Er is geen vlag `--verbose`. Gebruik `--dry-run` om een voorbeeld van geplande acties te bekijken,
`--json` voor machineleesbare resultaten en `openclaw update status --json`
alleen voor kanaal/beschikbaarheid. De breedsprakigheid van de Gateway-console (`--verbose`) en
het logniveau van bestanden (`logging.level: "debug"`/`"trace"`) zijn onafhankelijke instellingen; zie
[Gateway-logboekregistratie](/nl/gateway/logging).

<Note>
In Nix-modus (`OPENCLAW_NIX_MODE=1`) zijn wijzigende uitvoeringen van `openclaw update` uitgeschakeld. Werk in plaats daarvan de Nix-bron of flake-invoer voor deze installatie bij; gebruik voor nix-openclaw de agentgerichte [Snelstart](https://github.com/openclaw/nix-openclaw#quick-start). `openclaw update status` en `openclaw update --dry-run` blijven alleen-lezen.
</Note>

<Warning>
Downgrades vereisen bevestiging omdat oudere versies de configuratie kunnen beschadigen.
Als de installatie sessies al naar SQLite heeft gemigreerd, herstel dan de gearchiveerde verouderde
transcriptartefacten voordat je een oudere versie met bestandsopslag start. Zie
[Doctor: downgraden na SQLite-migratie van sessies](/nl/cli/doctor#downgrading-after-session-sqlite-migration).
</Warning>

## `update status`

Toon het actieve updatekanaal, de git-tag/branch/SHA (alleen broncode-checkouts)
en de beschikbaarheid van updates.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| Vlag                  | Standaard | Beschrijving                         |
| --------------------- | --------- | ------------------------------------ |
| `--json`              | `false` | Druk machineleesbare status-JSON af. |
| `--timeout <seconds>` | `3`     | Time-out voor controles.             |

Voor extended-stable-pakketinstallaties voert status dezelfde openbare selectie
en verificatie van het exacte pakket uit als een update op de voorgrond. Het kan
`ahead of extended-stable` rapporteren wanneer de geïnstalleerde versie nieuwer is. JSON-fouten
bevatten `registry.reason` (`selector_missing`, `selector_query_failed`,
`exact_package_mismatch` of `unsupported_git_channel`).

## `update repair`

Voer de afronding van de update opnieuw uit nadat het kernpakket al is gewijzigd, maar latere
reparatiewerkzaamheden niet correct zijn voltooid. Dit is het ondersteunde herstelpad wanneer
`openclaw update` het nieuwe kernpakket heeft geïnstalleerd, maar de synchronisatie van plugins na de kernupdate,
beheerde npm-pluginmetadata, het vernieuwen van het register of Doctor-reparatie niet
tot een consistente toestand zijn gekomen.

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| Vlag                                             | Beschrijving                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | Bewaar het updatekanaal van de kern vóór de reparatie. Voor extended-stable gebruiken geschikte officiële npm-plugins met standaard- of `latest`-intentie de exacte geïnstalleerde kernversie als doel. Extended-stable-reparatie wordt voor Git-checkouts geweigerd zonder de configuratie te wijzigen. |
| `--json`                                         | Druk machineleesbare JSON over de afronding af.                                                                                                                                                                                                                      |
| `--timeout <seconds>`                            | Time-out voor reparatiestappen. Standaard `1800`.                                                                                                                                                                                                                |
| `--yes`                                          | Sla bevestigingsvragen over.                                                                                                                                                                                                                                         |
| `--acknowledge-clawhub-risk`                     | Hetzelfde gedrag als bij `openclaw update`.                                                                                                                                                                                                                         |
| `--no-restart`                                   | Geaccepteerd voor consistentie; reparatie start de Gateway nooit opnieuw.                                                                                                                                                                                            |

`update repair` voert `openclaw doctor --fix` uit, laadt de gerepareerde configuratie en
installatierecords opnieuw, synchroniseert bijgehouden plugins voor het actieve updatekanaal, werkt
beheerde npm-plugininstallaties bij, repareert ontbrekende geconfigureerde pluginpayloads,
vernieuwt het pluginregister en schrijft consistente metadata voor installatierecords.
Het installeert geen nieuw kernpakket en start de Gateway niet opnieuw.

## `update wizard`

Interactieve flow om een updatekanaal te kiezen en te bevestigen of de
Gateway daarna opnieuw moet worden gestart (standaard wordt opnieuw gestart). Als je `dev` selecteert zonder een git-
checkout, wordt aangeboden er een te maken.

| Vlag                  | Standaard | Beschrijving                    |
| --------------------- | --------- | ------------------------------- |
| `--timeout <seconds>` | `1800`  | Time-out voor elke updatestap. |

## Wat het doet

Door expliciet van kanaal te wisselen (`--channel ...`) blijft ook de installatiemethode
afgestemd:

- `dev` -> zorgt voor een git-checkout (standaard `~/openclaw`, of
  `$OPENCLAW_HOME/openclaw` wanneer `OPENCLAW_HOME` is ingesteld; overschrijf met
  `OPENCLAW_GIT_DIR`), werkt deze bij en installeert de globale CLI vanuit die
  checkout.
- `stable` -> installeert vanuit npm met `latest`.
- `extended-stable` -> verwerkt de openbare npm-selector `extended-stable`,
  verifieert het exact geselecteerde pakket en installeert die exacte versie. Er
  wordt niet teruggevallen op een andere selector en deze optie wordt voor Git-checkouts geweigerd.
- `beta` -> geeft de voorkeur aan npm-dist-tag `beta` en valt terug op `latest` wanneer de bèta
  ontbreekt of ouder is dan de huidige stabiele release.

### Overdracht bij opnieuw starten

De automatische updater van de Gateway-kern (wanneer ingeschakeld via de configuratie) start het CLI-
updatepad buiten de actieve aanvraaghandler van de Gateway. Control-plane-
pakketbeheerupdates met `update.run` en bewaakte updates van git-checkouts gebruiken
dezelfde overdracht via de beheerde service in plaats van de pakketstructuur te vervangen of
`dist/` opnieuw op te bouwen binnen het actieve Gateway-proces: de Gateway start een
losgekoppeld hulpproces en sluit af, waarna dat hulpproces `openclaw update --yes --json`
uitvoert van buiten de processtructuur van de Gateway. Als de overdracht niet beschikbaar is,
retourneert `update.run` een gestructureerd antwoord met de veilige shell-opdracht die
handmatig moet worden uitgevoerd.

Opgeslagen extended-stable-selecties ontvangen alleen-lezen hints bij het opstarten en elke 24 uur voor updates
wanneer `update.checkOnStart` is ingeschakeld. Deze controles passen nooit een update toe,
starten geen overdracht, herstarten de Gateway niet, gebruiken geen stabiele vertraging/jitter en
gebruiken niet het pollinginterval van bèta. Expliciete updates op de voorgrond, kale updates op de voorgrond met
opgeslagen `update.channel: "extended-stable"`, status op aanvraag en de bijbehorende beheerde
Gateway-overdracht blijven ondersteund.

Wanneer een lokale beheerde Gateway-service is geïnstalleerd en herstarten is ingeschakeld,
stoppen updates via de pakketbeheerder en vanuit een Git-checkout de actieve service voordat
ze de pakketstructuur vervangen of de checkout/builduitvoer wijzigen. Het updateprogramma
vernieuwt vervolgens de servicemetadata, herstart de service en verifieert de
herstarte Gateway voordat `Gateway: restarted and verified.` wordt gemeld.
Updates via de pakketbeheerder verifiëren daarnaast dat de herstarte Gateway de
verwachte pakketversie meldt; updates vanuit een Git-checkout verifiëren na de rebuild
de gezondheid van de Gateway en de gereedheid van de service.

Updates via de pakketbeheerder blijven normaal gesproken de Node-binary gebruiken die in de
beheerde service is vastgelegd. Als die Node de doelrelease niet kan uitvoeren, maar de huidige
CLI-Node dat wel kan en aantoonbaar is dat de service bij het bijgewerkte pakket hoort,
gebruikt een update waarbij herstarten is ingeschakeld de huidige Node voor de afronding en herschrijft
de servicemetadata naar die runtime. `--no-restart` kan servicemetadata niet herstellen,
dus bij dezelfde niet-overeenkomende runtime wordt gestopt voordat het pakket wordt gewijzigd.

Op macOS verifieert de controle na de update ook of de LaunchAgent voor het
actieve profiel is geladen/actief is en of de geconfigureerde loopbackpoort
gezond is. Als de plist is geïnstalleerd maar launchd er geen toezicht op houdt, bootstrapt OpenClaw
de LaunchAgent automatisch opnieuw en voert het de controles voor gezondheid/versie/
kanaalgereedheid opnieuw uit (een nieuwe bootstrap laadt de taak `RunAtLoad` rechtstreeks,
zodat het herstel de nieuw gestarte Gateway niet onmiddellijk `kickstart -k`). Als
de Gateway nog steeds niet gezond wordt, wordt de opdracht afgesloten met een niet-nulstatus en
worden het pad naar het herstartlogboek en instructies voor herstarten, opnieuw installeren en
het terugdraaien van het pakket weergegeven.

Als herstarten niet kan worden uitgevoerd, geeft de opdracht `Gateway: restart skipped (...)` of
`Gateway: restart failed: ...` weer met een handmatige hint voor `openclaw gateway restart`.
Met `--no-restart` wordt het pakket nog steeds vervangen of de Git-rebuild nog steeds uitgevoerd, maar de
beheerde service wordt niet gestopt of herstart, waardoor de actieve Gateway oude
code blijft gebruiken totdat je deze handmatig herstart.

### Vorm van het antwoord van het besturingsvlak

Wanneer `update.run` via het Gateway-besturingsvlak wordt uitgevoerd bij een installatie
via een pakketbeheerder of een bewaakte Git-checkout, rapporteert de handler het initiëren van de overdracht
afzonderlijk van de CLI-update die doorgaat nadat de Gateway is afgesloten:

- `ok: true`, `result.status: "skipped"`,
  `result.reason: "managed-service-handoff-started"` en
  `handoff.status: "started"`: de Gateway heeft de overdracht naar de beheerde service gemaakt
  en zijn eigen herstart gepland, zodat de losgekoppelde helper
  `openclaw update --yes --json` buiten het actieve serviceproces kan uitvoeren.
- `ok: false`, `result.reason: "managed-service-handoff-unavailable"` en
  `handoff.status: "unavailable"`: OpenClaw kon geen bewakende
  servicegrens en duurzame service-identiteit vinden voor een veilige overdracht (voor
  systemd-overdracht is bijvoorbeeld de eenheidsidentiteit `OPENCLAW_SYSTEMD_UNIT` vereist,
  niet alleen aanwezige systemd-procesmarkeringen). Het antwoord bevat
  `handoff.command`, de shellopdracht die buiten de Gateway moet worden uitgevoerd.
- `ok: false`, `result.reason: "managed-service-handoff-failed"`: de Gateway
  probeerde de overdracht te maken, maar kon de losgekoppelde helper niet starten.

De payload `sentinel` wordt geschreven voordat de Gateway wordt afgesloten, en de CLI-
overdracht werkt diezelfde herstartsentinel bij nadat de gezondheidscontroles na de herstart van de beheerde service
zijn voltooid. Tijdens de overdracht kan de sentinel
`stats.reason: "restart-health-pending"` bevatten zonder succesvolle voortzetting; de
herstarte Gateway pollt deze en start de voortzetting pas nadat de CLI
de gezondheid van de service heeft geverifieerd en de sentinel heeft herschreven met het uiteindelijke resultaat `ok`.
`openclaw status` en `openclaw status --all` tonen een rij `Update restart`
zolang die sentinel in behandeling of mislukt is, en `update.status` vernieuwt en
retourneert de nieuwste sentinel.

## Flow voor Git-checkouts

### Kanaalselectie

- `stable`: check de nieuwste niet-bèta-tag uit en voer daarna de build en doctor uit.
- `beta`: geef de voorkeur aan de nieuwste tag `-beta` en val terug op de nieuwste stabiele tag
  wanneer bèta ontbreekt of ouder is.
- `dev`: check `main` uit en voer daarna fetch en rebase uit.
- `extended-stable`: niet ondersteund voor Git-checkouts; de checkout wordt niet
  gewijzigd.

### Updatestappen

<Steps>
  <Step title="Schone worktree verifiëren">
    Vereist dat er geen niet-gecommitte wijzigingen zijn.
  </Step>
  <Step title="Van kanaal wisselen">
    Schakelt over naar het geselecteerde kanaal (tag of branch).
  </Step>
  <Step title="Upstream ophalen">
    Alleen voor dev.
  </Step>
  <Step title="Voorafgaande buildcontrole (alleen dev)">
    Voert de TypeScript-build uit in een tijdelijke worktree. Als de tip mislukt, wordt tot 10 commits teruggegaan om de nieuwste bouwbare commit te vinden. Stel `OPENCLAW_UPDATE_PREFLIGHT_LINT=1` in om tijdens deze voorafgaande controle ook lint uit te voeren; lint wordt in beperkte seriële modus uitgevoerd omdat hosts voor gebruikersupdates vaak kleiner zijn dan CI-runners.
  </Step>
  <Step title="Rebase uitvoeren">
    Voert een rebase uit op de geselecteerde commit (alleen dev).
  </Step>
  <Step title="Afhankelijkheden installeren">
    Gebruikt de pakketbeheerder van de repo. Voor pnpm-checkouts bootstrapt het updateprogramma `pnpm` op aanvraag (eerst via `corepack` en daarna via een tijdelijke fallback naar `npm install pnpm@11`) in plaats van `npm run build` binnen een pnpm-workspace uit te voeren. Als het bootstrappen van pnpm nog steeds mislukt, stopt het updateprogramma vroegtijdig met een pakketbeheerder-specifieke fout in plaats van `npm run build` in de checkout te proberen.
  </Step>
  <Step title="Control UI bouwen">
    Bouwt de Gateway en de Control UI.
  </Step>
  <Step title="Doctor uitvoeren">
    `openclaw doctor` wordt uitgevoerd als de laatste controle voor een veilige update.
  </Step>
  <Step title="Plugins synchroniseren">
    Synchroniseert plugins met het actieve kanaal. Dev gebruikt gebundelde plugins; stable en beta gebruiken npm. Werkt bijgehouden plugininstallaties bij.
  </Step>
</Steps>

### Details van pluginsynchronisatie

Op het bètakanaal proberen bijgehouden npm- en ClawHub-plugininstallaties die de
standaard-/nieuwste lijn volgen eerst een pluginrelease `@beta`. Als de plugin geen
bètarelease heeft, valt OpenClaw terug op de vastgelegde standaard-/nieuwste specificatie en
meldt het een waarschuwing. Voor npm-plugins valt OpenClaw ook terug wanneer het bètapakket
bestaat maar de installatievalidatie mislukt. Deze fallbackwaarschuwingen
laten de kernupdate niet mislukken. Exacte versies en expliciete tags worden nooit herschreven.

<Warning>
Als een exact vastgezette npm-pluginupdate resulteert in een artefact waarvan de integriteit afwijkt van de opgeslagen installatievermelding, breekt `openclaw update` de update van dat pluginartefact af in plaats van het te installeren. Installeer de plugin alleen opnieuw of werk deze alleen expliciet bij nadat je hebt geverifieerd dat je het nieuwe artefact vertrouwt.
</Warning>

<Note>
Mislukkingen van de pluginsynchronisatie na de update die beperkt zijn tot een beheerde plugin en die door het synchronisatiepad kunnen worden omzeild (bijvoorbeeld een onbereikbaar npm-register voor een niet-essentiële plugin), worden als waarschuwingen gemeld nadat de kernupdate is geslaagd. Het JSON-resultaat behoudt op het hoogste niveau update-`status: "ok"` en meldt `postUpdate.plugins.status: "warning"` met richtlijnen voor `openclaw update repair` en `openclaw plugins inspect <id> --runtime --json`. Onverwachte uitzonderingen in het updateprogramma of de synchronisatie laten het updateresultaat nog steeds mislukken. Herstel de installatie- of updatefout van de plugin en voer daarna `openclaw update repair` opnieuw uit. Wanneer een mislukte update een beheerde plugin onbruikbaar achterlaat, schakelt OpenClaw de runtimevermelding ervan uit en stelt het actieve slots opnieuw in zonder het door de operator opgestelde beleid `plugins.allow` of `plugins.deny` te wijzigen.

Na de synchronisatiestap per plugin voert `openclaw update` vóór de herstart van de Gateway een verplichte **convergentie na de kernupdate** uit: ontbrekende geconfigureerde pluginpayloads worden hersteld, elke _actieve_ bijgehouden installatievermelding op schijf wordt gevalideerd en er wordt statisch geverifieerd dat de bijbehorende `package.json` kan worden geparseerd (en dat een eventueel expliciet gedeclareerde `main` bestaat). Mislukkingen in deze controle en een ongeldige configuratiesnapshot retourneren `postUpdate.plugins.status: "error"` en wijzigen update-`status` op het hoogste niveau in `"error"`, zodat `openclaw update` wordt afgesloten met een niet-nulstatus en de Gateway _niet_ opnieuw wordt gestart met een niet-geverifieerde pluginset. De fout bevat gestructureerde regels `postUpdate.plugins.warnings[].guidance` die verwijzen naar `openclaw update repair` en `openclaw plugins inspect <id> --runtime --json`. Uitgeschakelde pluginvermeldingen en vermeldingen die geen aan een vertrouwde bron gekoppelde officiële synchronisatiedoelen zijn, worden hier overgeslagen (overeenkomstig het beleid `skipDisabledPlugins` dat door de controle op ontbrekende payloads wordt gebruikt), zodat een verouderde vermelding van een uitgeschakelde plugin een verder geldige update niet kan blokkeren.

Wanneer de bijgewerkte Gateway start, is het laden van plugins alleen-verifiëren: tijdens het opstarten worden geen pakketbeheerders uitgevoerd en worden afhankelijkheidsstructuren niet gewijzigd. Herstarts via pakketbeheerder-`update.run` worden overgedragen aan het CLI-pad voor beheerde services, zodat het pakket buiten het oude Gateway-proces wordt vervangen en de gezondheidscontroles van de service bepalen of de update als voltooid kan worden gemeld.
</Note>

Nadat een extended-stable-kernupdate is geslaagd, richten de integriteitscontrole en
convergentie van plugins na de kernupdate zich op in aanmerking komende officiële npm-plugins met exact de
geïnstalleerde kernversie. Voor standaard-/`latest`-intentie vraagt OpenClaw
plugin-`@extended-stable` niet op en valt het niet terug op npm-`latest`; de pakketversie
wordt afgeleid van de geïnstalleerde kern. Expliciet vastgezette versies, expliciete tags anders dan `latest`,
pakketten van derden en niet-npm-bronnen behouden hun bestaande intentie.

Voor installaties via een pakketbeheerder bepaalt `openclaw update` de doelpakketversie
voordat de pakketbeheerder wordt aangeroepen. Globale npm-installaties gebruiken een gefaseerde
installatie: OpenClaw installeert het nieuwe pakket in een tijdelijk npm-prefix,
laat het kandidaatpakket tijdens `preinstall` de Node-versie van de host valideren
en verifieert daar de verpakte inventaris `dist`. Een verpakte voltooiingsbeveiliging
blijft buiten die inventaris totdat `preinstall` slaagt, zodat pakketbeheerders
die levenscyclusscripts overslaan ook vóór activering stoppen. Op npm 12 en nieuwer
staat het updateprogramma alleen de levenscyclus van kandidaat OpenClaw toe; scripts van
transitieve afhankelijkheden blijven geblokkeerd. OpenClaw vervangt vervolgens de schone pakketstructuur
in het werkelijke globale prefix. Als de verificatie mislukt, worden doctor na de update,
pluginsynchronisatie en herstartwerkzaamheden niet vanuit de verdachte structuur uitgevoerd. Zelfs wanneer de
geïnstalleerde versie al overeenkomt met het doel, vernieuwt de opdracht de
globale pakketinstallatie en voert daarna pluginsynchronisatie, vernieuwing van de voltooiing
voor kernopdrachten en herstartwerkzaamheden uit. Hierdoor blijven verpakte sidecars en door kanalen beheerde
pluginvermeldingen afgestemd op de geïnstalleerde OpenClaw-build, terwijl volledige
vernieuwingen van voltooiing voor pluginopdrachten worden overgelaten aan expliciete
uitvoeringen van `openclaw completion --write-state`.

## Gerelateerd

- `openclaw doctor` (biedt aan om bij Git-checkouts eerst een update uit te voeren)
- [Ontwikkelingskanalen](/nl/install/development-channels)
- [Bijwerken](/nl/install/updating)
- [CLI-referentie](/nl/cli)
