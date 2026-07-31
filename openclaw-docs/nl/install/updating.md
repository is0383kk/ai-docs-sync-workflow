---
read_when:
    - OpenClaw bijwerken
    - Er gaat iets mis na een update
summary: OpenClaw veilig bijwerken (globale installatie of broncode), plus terugdraaistrategie
title: Bijwerken
x-i18n:
    generated_at: "2026-07-27T05:09:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

Houd OpenClaw up-to-date.

Zie voor het vervangen van Docker-, Podman- en Kubernetes-images
[Container-images upgraden](/nl/install/docker#upgrading-container-images). De
Gateway voert vóór de gereedheidscontrole upgradebewerkingen uit die veilig zijn bij het opstarten en sluit af als gekoppelde
status handmatig moet worden hersteld.

## Aanbevolen: `openclaw update`

Detecteert je installatietype (npm, pnpm, Bun of git), haalt de nieuwste versie op, voert `openclaw doctor` uit en herstart de Gateway.

```bash
openclaw update
```

Wissel van kanaal of kies een specifieke versie:

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # voorbeeldweergave zonder toe te passen
```

`openclaw update` heeft geen vlag `--verbose` (het installatieprogramma wel). Gebruik voor diagnostiek
`--dry-run` om geplande acties vooraf te bekijken, `--json` voor gestructureerde resultaten of
`openclaw update status --json` om de kanaal- en beschikbaarheidsstatus te controleren.

`--channel beta` geeft de voorkeur aan de npm-dist-tag beta, maar valt terug op stable/latest
wanneer de beta-tag ontbreekt of de versie ervan ouder is dan de nieuwste stabiele
release. Gebruik in plaats daarvan `--tag beta` voor een eenmalige pakketupdate die is vastgezet op de onbewerkte npm-
dist-tag beta.

`--channel extended-stable` geldt alleen voor pakketten en de installatie blijft
uitsluitend op de voorgrond plaatsvinden. OpenClaw leest de openbare npm-selector `extended-stable`,
verifieert het exact geselecteerde pakket en installeert die exacte versie. Ontbrekende
of inconsistente registergegevens leiden tot een veilige fout; er wordt nooit teruggevallen op `latest`.
Als de geselecteerde versie ouder is dan de geïnstalleerde versie, blijft de normale
bevestiging voor downgraden van toepassing. De CLI slaat het kanaal op na een
geslaagde kernupdate; een rechtstreekse `npm install -g openclaw@extended-stable`
werkt `update.channel` niet bij.
Na het vervangen van de kern worden in aanmerking komende officiële npm-plugins met een kale/standaardintentie of
`latest`-intentie afgestemd op die exacte kernversie. Exacte vastzettingen en expliciete
niet-`latest`-tags, plugins van derden en bronnen anders dan npm blijven ongewijzigd.
Catalogusinstallaties die door huidige OpenClaw-versies zijn gemaakt, behouden die standaard-
intentie. Oudere records die alleen een exacte versie bevatten, blijven vastgezet omdat
OpenClaw niet veilig het verschil kan bepalen tussen een oude automatische vastzetting en een gebruikersvastzetting; voer
`openclaw plugins update @openclaw/name` eenmaal uit op het extended-stable-kanaal
om die plugin weer exacte-kerntracking te laten gebruiken.

`--channel dev` biedt een permanente, meebewegende GitHub-`main`-checkout. Voor een eenmalige
pakketupdate wordt `--tag main` gekoppeld aan de pakketspecificatie `github:openclaw/openclaw#main`
en rechtstreeks via het doelpakketbeheer (npm/pnpm/bun) geïnstalleerd.

Voor beheerde plugins is een ontbrekende betarelease een waarschuwing, geen fout: de
kernupdate kan nog steeds slagen terwijl een plugin terugvalt op de vastgelegde
standaard-/nieuwste release.

Zie [Releasekanalen](/nl/install/development-channels) voor de semantiek van kanalen.

## Wisselen tussen npm- en git-installaties

Gebruik kanalen om het installatietype te wijzigen. De updater behoudt je status, configuratie,
inloggegevens en werkruimte in `~/.openclaw`; alleen de OpenClaw-
code-installatie die de CLI en Gateway gebruiken, wordt gewijzigd.

```bash
# npm-pakketinstallatie -> bewerkbare git-checkout
openclaw update --channel dev

# git-checkout -> npm-pakketinstallatie
openclaw update --channel stable
```

Bekijk eerst een voorbeeld van de wisseling van installatiemodus:

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` zorgt voor een git-checkout, bouwt deze en installeert de globale CLI vanuit die
checkout. De kanalen `stable`, `extended-stable` en `beta` gebruiken pakket-
installaties. Extended-stable wordt geweigerd bij een git-checkout zonder deze te wijzigen of
converteren. Als de Gateway al is geïnstalleerd, vernieuwt `openclaw update`
de servicemetadata en herstart deze, tenzij je `--no-restart` doorgeeft.

Voor pakketinstallaties met een beheerde Gateway-service richt `openclaw update` zich op
de pakketroot die door die service wordt gebruikt. Als de shellopdracht `openclaw` uit
een andere installatie komt, toont de updater beide roots en het Node-pad van de beheerde
service, en controleert deze de Node-versie aan de hand van de vereiste `engines.node`
van de doelrelease voordat het pakket wordt vervangen.

## Servers met een broncheckout (referentiescript)

Teams die een Gateway rechtstreeks vanuit een git-checkout op een server uitvoeren, kunnen deze bijwerken
met `scripts/update-gateway.sh` vanuit die checkout. Dit is de referentie
voor een efficiënte update van een bronserver: het herstelt bijgehouden builduitvoer die
`pnpm build` herschrijft, leidt tot een veilige fout bij alle andere lokale wijzigingen, fast-forwardt
`main` (of rebaset een lokale servertak op `origin/main`), installeert
afhankelijkheden, voert een schone build uit en herstart de Gateway.

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

Overschrijf de herstart voor aangepaste service-eenheden of sla deze volledig over:

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

Geef voor een eenvoudige broninstallatie voor één gebruiker de voorkeur aan `openclaw update --channel dev`
— deze beheert de checkout, build en herstart van de Gateway voor je.

## Alternatief: voer het installatieprogramma opnieuw uit

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Voeg `--no-onboard` toe om onboarding over te slaan. Geef
`--install-method git --no-onboard` of `--install-method npm --no-onboard` door om een specifiek installatietype af te dwingen.

Als `openclaw update` mislukt na de npm-pakketinstallatiefase, voer dan het
installatieprogramma opnieuw uit. Het roept de updater niet aan; het voert de globale pakket-
installatie rechtstreeks uit en kan een gedeeltelijk bijgewerkte npm-installatie herstellen.

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

Zet het herstel vast op een specifieke versie of dist-tag met `--version`:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## Alternatief: handmatig met npm, pnpm of bun

```bash
npm i -g openclaw@latest
```

Geef bij installaties onder toezicht de voorkeur aan `openclaw update`: dit kan de pakket-
vervanging coördineren met de actieve Gateway-service. Als je een installatie onder toezicht handmatig
bijwerkt, stop dan eerst de beheerde Gateway. Pakketbeheerders vervangen bestanden
ter plaatse en een actieve Gateway kan anders tijdens het vervangen proberen kern- of pluginbestanden
te laden. Herstart de Gateway nadat het pakketbeheer is voltooid, zodat deze
de nieuwe installatie gebruikt.

Als `openclaw update` bij een globale Linux-systeeminstallatie die eigendom is van root mislukt met
`EACCES`, herstel je met systeem-npm terwijl de Gateway gestopt blijft voor de
handmatige vervanging. Gebruik dezelfde profielvlaggen/-omgeving die je normaal voor
die Gateway gebruikt. Vervang `/usr/bin/npm` door de systeem-npm die eigenaar is van het
globale root-prefix op je host:

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

Verifieer vervolgens:

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

Wanneer `openclaw update` een globale npm-installatie beheert, installeert het doel
eerst in een tijdelijk npm-prefix. Het kandidaatpakket valideert de Node-
versie van de host tijdens `preinstall`; pas daarna verifieert OpenClaw de verpakte
`dist`-inventaris en vervangt het de schone pakketstructuur in het werkelijke globale prefix. Een
verpakte voltooiingsbeveiliging wordt weggelaten uit de verwachte inventaris en pas verwijderd
nadat `preinstall` slaagt, zodat overgeslagen levenscyclusscripts eveneens vóór de
vervanging mislukken. Bij npm 12 en nieuwer keurt de updater alleen de levenscyclus van
OpenClaw als kandidaat goed; scripts van transitieve afhankelijkheden blijven geblokkeerd. Dit voorkomt dat npm
een nieuw pakket over achtergebleven bestanden van het oude pakket heen legt. Als de installatie-
opdracht mislukt, probeert OpenClaw het eenmaal opnieuw met `--omit=optional`, wat nuttig is op hosts
waar native optionele afhankelijkheden niet kunnen worden gecompileerd.

Door OpenClaw beheerde npm-update- en plugin-updateopdrachten wissen ook npm's
`min-release-age`-toeleveringsketenquarantaine (of de oudere configuratiesleutel `before`)
voor het onderliggende npm-proces. Dat beleid bestaat voor algemene bescherming, maar een
expliciete OpenClaw-update betekent: "installeer de geselecteerde release nu."

```bash
pnpm add -g openclaw@latest
```

Als pnpm 11 OpenClaw 2026.7.1 heeft geïnstalleerd, voer die handmatige opdracht dan eenmaal uit. Die
release dateert van vóór de geïsoleerde globale pakketindeling van pnpm 11, waardoor de updater
een andere npm-installatie kan aanzien voor de actieve CLI. Latere releases behouden
het eigenaarschap van pnpm en volgen tijdens updates de pakketroot van het vervangende pakket. Ze
gebruiken ook de door de eigenaar-beheerder gerapporteerde globale bin-directory en stoppen vóór
wijziging wanneer de beschikbare pnpm-opdracht een andere globale root of hoofdversie meldt,
of wanneer het aanroepende pakket verweesd is of daar niet de enige actieve OpenClaw-
installatie is.

Als OpenClaw een globale pnpm 11-installatiegroep deelt met een ander pakket, stopt de
automatische updater voordat de groep wordt gewijzigd. Werk de oorspronkelijke
door komma's gescheiden groep handmatig bij, zodat de zusterpakketten en het buildbeleid
intact blijven.

```bash
bun add -g openclaw@latest
```

### Geavanceerde onderwerpen voor npm-installaties

<AccordionGroup>
  <Accordion title="Alleen-lezen pakketstructuur">
    OpenClaw behandelt verpakte globale installaties tijdens runtime als alleen-lezen, zelfs wanneer de globale pakketdirectory beschrijfbaar is voor de huidige gebruiker. Installaties van pluginpakketten bevinden zich in npm-/git-roots onder de gebruikersconfiguratiedirectory die eigendom zijn van OpenClaw, en bij het opstarten van de Gateway wordt de OpenClaw-pakketstructuur niet gewijzigd.

    Sommige Linux-npm-configuraties installeren globale pakketten in directory's die eigendom zijn van root, zoals `/usr/lib/node_modules/openclaw`. OpenClaw ondersteunt die indeling omdat opdrachten voor het installeren/bijwerken van plugins buiten die globale pakketdirectory schrijven.

  </Accordion>
  <Accordion title="Versterkte systemd-eenheden">
    Geef OpenClaw schrijftoegang tot de configuratie-/statusroots, zodat expliciete plugininstallaties, pluginupdates en opschoning door doctor hun wijzigingen kunnen opslaan:

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="Voorcontrole van schijfruimte">
    Vóór pakketupdates en expliciete plugininstallaties probeert OpenClaw naar beste vermogen de schijfruimte op het doelvolume te controleren. Weinig ruimte levert een waarschuwing met het gecontroleerde pad op, maar blokkeert de update niet omdat bestandssysteemquota, momentopnamen en netwerkvolumes na de controle kunnen veranderen. De daadwerkelijke installatie door het pakketbeheer en de verificatie na installatie blijven doorslaggevend.
  </Accordion>
</AccordionGroup>

## Automatische updater

Standaard uitgeschakeld. Schakel deze in via `~/.openclaw/openclaw.json`:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| Kanaal            | Gedrag                                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | Wordt toegepast na een ingebouwde vertraging met deterministische jitter voor een gespreide uitrol.                           |
| `extended-stable` | Controleert bij het opstarten en elke 24 uur op een alleen-lezen updatehint wanneer `checkOnStart` is ingeschakeld. Wordt nooit automatisch toegepast. |
| `beta`            | Controleert volgens een ingebouwd interval en past de update onmiddellijk toe.                                                |
| `dev`             | Geen automatische toepassing. Gebruik `openclaw update` handmatig.                                                           |

De Gateway registreert bij het opstarten ook een updatehint (uitschakelen met
`update.checkOnStart: false`). Opgeslagen selecties voor extended-stable gebruiken dit
alleen-lezenhintpad en het bestaande hintinterval van 24 uur, maar voeren nooit
automatische installatie, overdracht, herstart, stabiele vertraging/jitter of bètapolling uit.
Stel voor een downgrade of herstel na een incident `OPENCLAW_NO_AUTO_UPDATE=1` in de Gateway-omgeving in om automatische toepassingen te blokkeren, zelfs wanneer `update.auto.enabled` is geconfigureerd. Updatehints bij het opstarten kunnen nog steeds worden uitgevoerd, tenzij `update.checkOnStart` ook is uitgeschakeld.

Updates van de pakketbeheerder die via het live besturingsvlak van de Gateway
(`update.run`) worden aangevraagd, vervangen de pakketstructuur niet binnen het actieve Gateway-
proces. Bij installaties als beheerde service start de Gateway een losgekoppelde overdracht,
sluit af en laat het normale `openclaw update --yes --json`-CLI-pad de
service stoppen, het pakket vervangen, servicemetadata vernieuwen, opnieuw starten, de
Gateway-versie en bereikbaarheid verifiëren en waar mogelijk een geïnstalleerde maar niet-geladen macOS-
LaunchAgent herstellen. Als de Gateway die overdracht niet veilig kan uitvoeren,
rapporteert `update.run` een veilige shellopdracht in plaats van de pakketbeheerder
in het proces uit te voeren.

De updatekaart in de zijbalk van de Control UI toont **Gateway bijwerken** wanneer deze
deze `update.run`-flow rechtstreeks start. Dit omvat de in een browser gehoste Control UI, externe
Gateways en handmatig beheerde lokale Gateways.

In de ondertekende macOS-app verandert een lokale Gateway die eigendom is van de app die kaart in
**Mac-app + Gateway bijwerken**. Sparkle werkt eerst de app bij; na het opnieuw starten
voert de app `openclaw update --tag <app-version> --json` uit, herstart de Gateway
en verifieert de status in een voortgangsvenster in installatiestijl. Het venster verschijnt alleen
wanneer die beheerde Gateway een update, reparatie of installatie nodig heeft; updates die alleen de app betreffen, starten
rechtstreeks opnieuw in de app. Foutdetails blijven zichtbaar met de acties Opnieuw proberen, [Updatehandleiding](/nl/install/updating) en
[Discord](https://discord.gg/clawd). De app gebruikt dit gecoördineerde
pad nooit voor een externe of extern beheerde Gateway, voert nooit een downgrade van een nieuwere
Gateway uit en negeert nooit een `extended-stable`-kanaalvastlegging.

Wanneer de update slaagt, zet de app een eenmalige welkomstgebeurtenis in de wachtrij voor de
meest recente directe sessie op het hoogste niveau met een echte gebruikers-/kanaalinteractie. Cron-uitvoeringen,
Heartbeats en sessie-updates die alleen op de achtergrond plaatsvinden, wijzigen die selectie niet. In de
externe modus werkt de app alleen de runtime van de lokale Mac-node bij en verzendt de gebeurtenis
alleen wanneer de verbonden externe Gateway minstens even nieuw is als de app.

## Na het bijwerken

<Steps>

### Doctor uitvoeren

```bash
openclaw doctor
```

Migreert de configuratie, controleert DM-beleid en controleert de status van de Gateway. Details: [Doctor](/nl/gateway/doctor)

### De Gateway opnieuw starten

```bash
openclaw gateway restart
```

### Verifiëren

```bash
openclaw health
```

</Steps>

## Terugdraaien

Terugdraaien bestaat uit twee lagen:

1. Installeer oudere OpenClaw-code opnieuw met behoud van de huidige status.
2. Herstel de status van vóór de update alleen wanneer de oudere code een gemigreerde
   configuratie of database niet kan gebruiken.

Begin met het terugdraaien van alleen de code. Bij het herstellen van de status gaan wijzigingen verloren die na
de back-up zijn aangebracht.

### Vóór het bijwerken: een geverifieerde back-up maken

`openclaw update` bewaart automatisch een kopie van de configuratie van vóór de update, maar maakt geen
volledig herstelpunt voor de status. Maak vóór een belangrijke update expliciet
een herstelpunt:

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

Het archiefmanifest vermeldt de OpenClaw-versie en de bronpaden die
in de back-up zijn opgenomen. Het archief kan referenties, authenticatieprofielen en kanaalstatus
bevatten. Bewaar het daarom met alleen-eigenaarrechten en dezelfde bescherming als de
map met de actieve status. Zie [Back-up](/nl/cli/backup) voor opgenomen en bewust
weggelaten bestanden.

Voor een byte-voor-byteherstelpunt dat vluchtige artefacten bevat die uit
het overdraagbare archief zijn weggelaten, stop je de Gateway en gebruik je een door je platform
geleverde momentopname van het bestandssysteem, volume of de virtuele machine.

### Een pakketinstallatie terugdraaien

Geef gepubliceerde versies weer en bekijk en installeer vervolgens de bekende goede versie:

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

`openclaw update --tag` heeft de voorkeur boven een rechtstreekse installatie via de pakketbeheerder. Deze
detecteert de downgrade, vraagt om bevestiging, voert convergentie van beheerde Plugins
en compatibiliteitscontroles voor het geïnstalleerde doel uit, vernieuwt service-
metadata, herstart de Gateway en verifieert de actieve versie. Als het opgeslagen
kanaal `extended-stable` is, gebruik je
`--channel stable --tag <known-good-version>`, omdat exacte eenmalige tags niet
met de selector `extended-stable` kunnen worden gecombineerd.

Pakketupdates worden vóór activering klaargezet en geverifieerd. Als het
wisselen van het bestandssysteem of het vervangen van de opdrachtsnelkoppeling mislukt, herstelt OpenClaw automatisch het oude
pakket. Als na een geslaagde wissel een latere statuscontrole van de Gateway
mislukt, worden de vorige versie en instructies voor handmatig terugdraaien gemeld in plaats van
het pakket opnieuw automatisch te vervangen.

Als het CLI-updatepad niet beschikbaar is, gebruik je dezelfde pakketbeheerder en hetzelfde installatie-
bereik die eigenaar zijn van de huidige Gateway:

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

Vervang `npm` door `pnpm` of `bun` wanneer die beheerder eigenaar is van de installatie. Voorkom tijdens
herstel na een incident dat een ingeschakelde automatische updater onmiddellijk een
nieuwere release toepast door `OPENCLAW_NO_AUTO_UPDATE=1` in de Gateway-omgeving in te stellen.

### Een broncodecheckout terugdraaien

Gebruik een schone checkout en selecteer een bekende goede tag of commit:

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

Terugkeren naar de nieuwste versie: `git checkout main && git pull`.

De updater brengt een Git-checkout automatisch terug naar de vorige branch en
SHA wanneer de installatie van afhankelijkheden, build, UI-build of Doctor mislukt nadat een Git-
update is gestart. Handmatig uitchecken blijft vereist wanneer je bewust
een oudere commit kiest.

### Downgraden over de SQLite-migratie van sessies heen

Gebruik voordat je een oudere, bestandsgebaseerde OpenClaw-release start de huidige CLI om
gearchiveerde verouderde transcriptartefacten te herstellen:

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Hiermee worden geen SQLite-gegevens verwijderd. Sessies die na de SQLite-migratie zijn gemaakt,
bestaan alleen in SQLite en verschijnen niet in de oudere runtime. Zie
[Downgraden na de SQLite-migratie van sessies](/nl/cli/doctor#downgrading-after-session-sqlite-migration).

### Herstel de status alleen wanneer dat nodig is

Als de oudere code een nieuwere configuratie of een nieuwer databaseschema niet kan lezen, stop je de
Gateway en herstel je de geverifieerde momentopname van het bestandssysteem, volume of de virtuele machine van vóór de update.
Bewaar de huidige status afzonderlijk voordat je herstelt, omdat hierdoor
wijzigingen worden verwijderd die na de momentopname zijn aangebracht.

Brede `openclaw backup create`-archieven ondersteunen aanmaken en verifiëren, maar
niet het ter plaatse activeren van het volledige archief. Pak een breed archief uit in een tijdelijke
map en gebruik de `manifest.json`-toewijzing van bron naar archief voor offline
herstel. `openclaw backup sqlite restore` schrijft eveneens een geverifieerde database
naar een nieuw doel; het activeren van dat doel blijft een expliciete offlinebeheerders-
stap.

### Het terugdraaien verifiëren

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## Als je vastloopt

- Voer `openclaw doctor` opnieuw uit en lees de uitvoer zorgvuldig.
- Voor `openclaw update --channel dev` bij broncodecheckouts initialiseert de updater `pnpm` indien nodig automatisch. Als je een initialisatiefout van pnpm/corepack ziet, installeer je `pnpm` handmatig (of schakel je `corepack` opnieuw in) en voer je de update opnieuw uit.
- Bekijk: [Probleemoplossing](/nl/gateway/troubleshooting)
- Vraag het in Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## Gerelateerd

- [Installatieoverzicht](/nl/install): alle installatiemethoden.
- [Doctor](/nl/gateway/doctor): statuscontroles na updates.
- [Migreren](/nl/install/migrating): migratiehandleidingen voor hoofdversies.
