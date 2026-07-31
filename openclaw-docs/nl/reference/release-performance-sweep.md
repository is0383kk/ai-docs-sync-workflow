---
read_when:
    - Je valideert de opschoning van de prestaties en pakketgrootte van mei 2026
    - Je hebt de cijfers achter de blogpost over de prestaties en afhankelijkheden van OpenClaw nodig
    - Je wijzigt releasepoorten, package-shrinkwrap of afhankelijkheidsgrenzen van plugins
summary: Visueel overzicht en technisch bewijs voor de opschoning van prestaties, pakketgrootte, afhankelijkheden en shrinkwrap in mei 2026
title: Sweep van releaseprestaties
x-i18n:
    generated_at: "2026-07-27T05:21:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e98ffc9d63e14e078a19368917eb4278695e1426048dc21942f928af145d5e1
    source_path: reference/release-performance-sweep.md
    workflow: 16
---

Deze pagina legt het bewijsmateriaal vast achter de opschoning van OpenClaw in mei 2026 op het gebied van prestaties,
pakketgrootte, afhankelijkheden en shrinkwrap. Dit is de technische aanvulling
op de openbare blogpost.

Hier worden twee audits gecombineerd:

- **Prestatiecontrole van releases:** GitHub Releases vanaf `v2026.5.28` terug tot en met
  de stabiele `v2026.4.23`, met behulp van de `OpenClaw Performance`-workflow,
  `profile=smoke`, mock-providertraject. De meeste tagrijen bevatten één meting; de
  rijen `v2026.5.27` en `v2026.5.28` gebruiken de nieuwste artefacten met 3 herhalingen van de
  releasetakken.
- **Eerdere context uit april:** gepubliceerde mock-provider-
  basiswaarden van `clawgrit-reports` van `v2026.4.1` tot en met `v2026.5.2`, uitsluitend gebruikt om te voorkomen dat
  de defecte releases van eind april als openbare prestatiebasiswaarde worden beschouwd.
- **Controle van de installatieomvang:** nieuwe installaties van `npm install --ignore-scripts`
  in tijdelijke pakketten, met `du -sk node_modules` voor de grootte en een
  `node_modules`-doorloop voor het aantal pakketinstanties.
- **Controle van de npm-pakketgrootte:** `npm pack openclaw@<version> --dry-run --json`
  voor gepubliceerde releases, waarbij de gecomprimeerde tarballgrootte, uitgepakte grootte en
  het aantal bestanden zijn vastgelegd.

<Warning>
De belangrijkste prestatiecontrole gebruikt één rooktestmeting per tag, behalve de
rijen `v2026.5.27` en `v2026.5.28`, die de nieuwste artefacten met 3 herhalingen
van de releasetakken gebruiken. De eerdere context uit april gebruikt gepubliceerde medianen met 3 herhalingen
uit `clawgrit-reports`. Beschouw de cijfers als bewijs voor trends en
signalen om regressies op te sporen, niet als statistieken voor releasepoorten.
</Warning>

## Momentopname

Prestatiedekking: **77 aangevraagde releases**, **74 door artefacten onderbouwde punten**
en **3 niet-beschikbare CI-uitvoeringen**. Nieuwste gemeten stabiele punt: `v2026.5.28`.

<CardGroup cols={2}>
  <Card title="Stabiele agentbeurt" icon="gauge">
    **5,1x snellere koude beurt**

    - `v2026.4.14`: 9,8s
    - `v2026.5.28`: 1,9s

  </Card>
  <Card title="Gepubliceerd pakket" icon="package">
    **Tarball van 17,9MB**

    Nieuwste stabiele pakket, gedaald ten opzichte van de piek van 43,3MB voor de pakketgrootte in maart.

  </Card>
  <Card title="Nieuwste stabiele installatie" icon="hard-drive">
    **Nieuwe installatie van 361,7MiB**

    Verkleint de geneste afhankelijkheidsboom van OpenClaw sterk ten opzichte van de
    piek bij de introductie van shrinkwrap in `2026.5.22`, hoewel in de lokale
    installatieaudit nog steeds een kleinere geneste boom van 259,7MiB overblijft.

  </Card>
  <Card title="Afhankelijkhedengraaf" icon="boxes">
    **300 geïnstalleerde pakketten**

    Gemeten als unieke pakketnaam-/versiewortels in een nieuwe installatie met
    uitgeschakelde scripts; 71 wortels minder dan de vorige stabiele release.

  </Card>
</CardGroup>

## Wat er in 5.28 is gewijzigd

De opschoning tussen `v2026.5.27` en `v2026.5.28` verkleinde de graaf
voor de standaardinstallatie in plaats van de mogelijkheden zelf te verwijderen.

<CardGroup cols={2}>
  <Card title="Standaardgraaf op hoofdniveau" icon="git-branch">
    Het aantal unieke pakketnaam-/versiewortels daalde van **371** naar **300**. Het aantal pakketinstanties
    daalde van **372** naar **301**.
  </Card>
  <Card title="Geneste boom" icon="unplug">
    De geneste `openclaw/node_modules` daalde in dezelfde lokale installatieaudit van **656,1MiB** naar **259,7MiB**.
  </Card>
  <Card title="Native optionele afhankelijkheidskegels" icon="cpu">
    De platformoverstijgende native pakketkegel van `@napi-rs/canvas` kwam niet langer in
    de standaardinstallatie terecht.
  </Card>
  <Card title="Oppervlak van de toeleveringsketen" icon="shield">
    Minder standaardpakketten betekent minder tarballs, beheerders, native binaire bestanden,
    installatiegedrag en transitieve updatepaden die standaard moeten worden vertrouwd.
  </Card>
</CardGroup>

<Tip>
Shrinkwrap was op zichzelf niet het probleem. De slechte pakketvorm was dat wel.
`v2026.5.28` levert nog steeds shrinkwrap mee, maar de geneste afhankelijkheidsboom is veel
kleiner en de platformoverstijgende canvas-uitwaaiering is verdwenen in de lokale audit.
</Tip>

## Belangrijkste cijfers

Gebruik de defecte rijen van eind april niet als openbare prestatiebasiswaarden.
`v2026.4.23` en `v2026.4.29` zijn nuttig bewijs voor regressies, maar de grote
verschillen in de stijl van `14x` beschrijven voornamelijk het herstel van een slechte releasereeks.

Gebruik voor het verhaal van de blog de eerder in april gepubliceerde basiswaarde als schaal.
De basiswaarde is `v2026.4.14` uit de gepubliceerde mock-provideruitvoering van `clawgrit-reports`
(3 herhalingen; die uitvoering mislukte alleen omdat de diagnostische
tijdlijn niet werd uitgestuurd, waardoor de medianen voor koud, warm en RSS nog steeds bruikbaar zijn
als globale schaal). Beschouw dit als verhalende context, niet als statistiek voor een
releasepoort.

| Metriek          | Eerdere basiswaarde uit april | `v2026.5.28` |                    Verschil |
| --------------- | ---------------------: | -----------: | -----------------------: |
| Koude agentbeurt |                9,819ms |      1,908ms | 80,6% lager, 5,1x sneller |
| Warme agentbeurt |                7,458ms |      1,870ms | 74,9% lager, 4,0x sneller |
| Piek-RSS agent  |                686.2MB |      581.0MB |              15,3% lager |

Binnen de controle van mei veranderde de nieuwste rij van de releasetakken aanzienlijk ten opzichte van
`v2026.5.2`:

| Metriek          | `v2026.5.2` | `v2026.5.28` |       Verschil |
| --------------- | ----------: | -----------: | ----------: |
| Koude agentbeurt |     3,897ms |      1,908ms | 51,0% lager |
| Warme agentbeurt |     3,610ms |      1,870ms | 48,2% lager |
| Piek-RSS agent  |     613.7MB |      581.0MB |  5,3% lager |

Vergeleken met de vorige stabiele release:

| Metriek          | `v2026.5.27` | `v2026.5.28` |       Verschil |
| --------------- | -----------: | -----------: | ----------: |
| Koude agentbeurt |      2,231ms |      1,908ms | 14,5% lager |
| Warme agentbeurt |      2,226ms |      1,870ms | 16,0% lager |
| Piek-RSS agent  |      649.0MB |      581.0MB | 10,5% lager |

### Installatieomvang

| Metriek                                          |  Basiswaarde | `v2026.5.28` |       Verschil |
| ----------------------------------------------- | --------: | -----------: | ----------: |
| Installatiegrootte vanaf piek van `2026.5.22`              | 1,020.6MB |     361.7MiB | 64,6% lager |
| Installatiegrootte vanaf nieuwste release `2026.5.27`    |  767.1MiB |     361.7MiB | 52,8% lager |
| Afhankelijkheden vanaf maandpiek `2026.2.26`      |       645 |          300 | 53,5% lager |
| Afhankelijkheden vanaf nieuwste release `2026.5.27`    |       371 |          300 | 19,1% lager |
| Geneste `openclaw/node_modules` vanaf `2026.5.22` |   911.8MB |     259.7MiB | 71,5% lager |
| Geneste `openclaw/node_modules` vanaf `2026.5.27` |  656.1MiB |     259.7MiB | 60,4% lager |

### npm-pakketgrootte

| Versie     | Gecomprimeerde tarball | Uitgepakt pakket |  Bestanden | Opmerkingen                             |
| ----------- | -----------------: | ---------------: | -----: | --------------------------------- |
| `2026.1.30` |             12.8MB |           33.5MB |  4,607 | vroeg pakket met nieuwe merknaam           |
| `2026.2.26` |             23.6MB |           82.9MB | 10,125 | groei van functionaliteit                    |
| `2026.3.31` |             43.3MB |          182.6MB | 21,037 | hoogste punt van pakketgrootte           |
| `2026.4.29` |             22.9MB |           74.6MB |  9,309 | pakketopschoning zichtbaar           |
| `2026.5.12` |             23.4MB |           80.1MB | 12,035 | grote afsplitsing van externe plugins       |
| `2026.5.22` |             17.2MB |           76.9MB | 12,386 | documentatie/assets uitgesloten van pakket |
| `2026.5.27` |             17.8MB |           79.0MB | 12,509 | vorig stabiel pakket           |
| `2026.5.28` |             17.9MB |           81.0MB |  9,082 | nieuwste stabiele pakket             |

`2026.5.12` is de zichtbare mijlpaal voor Plugin-extractie in het wijzigingslogboek:
Amazon Bedrock, Bedrock Mantle, Slack, OpenShell-sandbox, Anthropic Vertex,
Matrix en WhatsApp zijn uit het kernpad voor afhankelijkheden verplaatst, zodat hun afhankelijkheidskegels
met die plugins worden geïnstalleerd in plaats van bij elke kerninstallatie.

## Samenvatting van Kova-agentbeurten

De stabiele reeks van april bevat twee verschillende verhalen. Eerder in april was deze traag,
maar herkenbaar. Eind april veranderde dit in een regressieafgrond. Bij `v2026.5.2`
daalt het mock-providertraject voor het eerst naar het bereik van 3-5s en begint het
consistent te slagen in de aangeleverde controle.

Eerdere gepubliceerde context:

| Release      | Kova | Koude beurt | Warme beurt | Piek-RSS agent |
| ------------ | ---- | --------: | --------: | -------------: |
| `v2026.4.10` | MISLUKT |  11,031ms |   7,962ms |        679.0MB |
| `v2026.4.12` | MISLUKT |  11,965ms |   8,289ms |        713.5MB |
| `v2026.4.14` | MISLUKT |   9,819ms |   7,458ms |        686.2MB |
| `v2026.4.20` | MISLUKT |  22,314ms |  18,811ms |        810.8MB |
| `v2026.4.22` | MISLUKT |   9,630ms |   7,459ms |        743.0MB |

Aangeleverde controle:

| Release             | Kova | Koude beurt | Warme beurt | Piek-RSS agent |
| ------------------- | ---- | --------: | --------: | -------------: |
| `v2026.4.23`        | MISLUKT |  47,847ms |   8,010ms |      1,082.7MB |
| `v2026.4.24`        | MISLUKT |  48,264ms |  25,483ms |        996.0MB |
| `v2026.4.25`        | MISLUKT |  81,080ms |  59,172ms |      1,113.9MB |
| `v2026.4.26`        | MISLUKT |  76,771ms |  54,941ms |      1,140.8MB |
| `v2026.4.27`        | MISLUKT |  60,902ms |  33,699ms |      1,156.0MB |
| `v2026.4.29`        | MISLUKT |  94,031ms |  57,334ms |      3,613.7MB |
| `v2026.5.2`         | GESLAAGD |   3,897ms |   3,610ms |        613.7MB |
| `v2026.5.7`         | GESLAAGD |   3,923ms |   3,693ms |        654.1MB |
| `v2026.5.12`        | GESLAAGD |   7,248ms |   6,629ms |        834.8MB |
| `v2026.5.18`        | GESLAAGD |   3,301ms |   2,913ms |        630.3MB |
| `v2026.5.20`        | GESLAAGD |   3,413ms |   2,952ms |        643.2MB |
| `v2026.5.22`        | GESLAAGD |   4,494ms |   4,093ms |        654.3MB |
| `v2026.5.26`        | GESLAAGD |   2,626ms |   2,282ms |        660.4MB |
| `v2026.5.27-beta.1` | GESLAAGD |   2,575ms |   2,217ms |        635.3MB |
| `v2026.5.27`        | GESLAAGD |   2,231ms |   2,226ms |        649.0MB |
| `v2026.5.28`        | GESLAAGD |   1,908ms |   1,870ms |        581.0MB |

## Bronmetingen

Bronmetingen zijn overgeslagen voor 17 geslaagde oudere refs, omdat die bronbomen
de vereiste meetingangspunten nog niet hadden. Voor die refs bestaan nog steeds
metrische gegevens voor agentbeurten.

Representatieve bronmeetpunten:

| Release             | Standaard `readyz` p50 | 50 plugins `readyz` p50 | CLI-status p50 | Max. RSS Plugin |
| ------------------- | -------------------: | ----------------------: | -------------: | -------------: |
| `v2026.4.29`        |              2,819ms |                 2,618ms |        1,679ms |        389.0MB |
| `v2026.5.2`         |              2,324ms |                 2,013ms |        1,384ms |        377.2MB |
| `v2026.5.7`         |              1,649ms |                 1,540ms |        1,175ms |        387.6MB |
| `v2026.5.18`        |              1,942ms |                 1,927ms |          607ms |        426.5MB |
| `v2026.5.20`        |              1,966ms |                 1,987ms |          621ms |        455.0MB |
| `v2026.5.22`        |              2,081ms |                 1,884ms |        5,095ms |        444.2MB |
| `v2026.5.26`        |              1,546ms |                 1,634ms |          656ms |        400.4MB |
| `v2026.5.27-beta.1` |              1,462ms |                 1,548ms |          548ms |        394.0MB |
| `v2026.5.27`        |              1,491ms |                 1,571ms |          553ms |        401.5MB |
| `v2026.5.28`        |              1,457ms |                 1,474ms |          623ms |        386.1MB |

De piek in CLI-status bij `v2026.5.22` is zichtbaar in deze tabel, hoewel het
agentbeurttraject nog steeds slaagde. Behoud de bronmetingen bij onderzoek naar
gerichte CLI- of Gateway-regressies.

## Audit van de installatieomvang

Afhankelijkheidsvoorbeelden gebruiken één stabiele release per maand, plus de
`2026.5.22`-gebeurtenis waarbij shrinkwrap werd geïntroduceerd en de nieuwste `2026.5.28`-release.

| Meetpunt           | Geïnstalleerde afhankelijkheden | Nieuwe installatie | OpenClaw-pakket | Geneste `openclaw/node_modules` | Root-shrinkwrap | Installatiegedrag van Canvas              |
| ------------------ | -------------------------------: | -----------------: | ---------------: | -----------------------------: | --------------- | ----------------------------------------- |
| Jan `2026.1.30`    |            605 |       438.4MB |           45.8MB |                          2.4MB | nee             | wrapper op het hoogste niveau + `darwin-arm64` |
| Feb `2026.2.26`    |            645 |       575.7MB |          110.1MB |                          3.5MB | nee             | wrapper op het hoogste niveau + `darwin-arm64` |
| Mrt `2026.3.31`    |            438 |       584.1MB |          234.8MB |                            0MB | nee             | wrapper op het hoogste niveau + `darwin-arm64` |
| Apr `2026.4.29`    |            392 |       335.0MB |           97.4MB |                            0MB | nee             | niets geïnstalleerd                       |
| `2026.5.22`        |            401 |     1,020.6MB |        1,020.4MB |                        911.8MB | ja              | genest: alle 12 `@napi-rs/canvas`-pakketten |
| Mei `2026.5.26`    |            371 |       767.5MB |          767.4MB |                        656.4MB | ja              | genest: alle 12 `@napi-rs/canvas`-pakketten |
| `2026.5.27`        |            371 |      767.1MiB |         766.9MiB |                       656.1MiB | ja              | genest: alle 12 `@napi-rs/canvas`-pakketten |
| Nieuwste `2026.5.28` |            300 |      361.7MiB |         361.6MiB |                       259.7MiB | ja              | niets geïnstalleerd                       |

### Shrinkwrap-grens

`2026.5.20` werd uitgebracht zonder root-shrinkwrap en zonder grote geneste
OpenClaw-afhankelijkheidsboom. `2026.5.22` introduceerde root-shrinkwrap en installeerde 911.8MB
onder de geneste `openclaw/node_modules`. `2026.5.28` behoudt shrinkwrap en installeert nog steeds
259.7MiB onder de geneste `openclaw/node_modules`, maar installeert niet langer
`@napi-rs/canvas`-pakketten in de lokale audit van een nieuwe installatie.

Inspectie van de gepubliceerde tarball bevestigt de grens:

| Versie      | Stabiel gepubliceerd? | Root-`npm-shrinkwrap.json` | Opmerkingen                               |
| ----------- | ---------------------- | -------------------------- | ----------------------------------------- |
| `2026.5.20` | ja                | nee                        | laatste stabiele release vóór shrinkwrap  |
| `2026.5.21` | nee               | n.v.t.                     | geen stabiele npm-release                 |
| `2026.5.22` | ja                | ja                         | shrinkwrap geïntroduceerd                 |
| `2026.5.23` | nee               | n.v.t.                     | geen stabiele npm-release                 |
| `2026.5.24` | nee               | n.v.t.                     | geen stabiele npm-release                 |
| `2026.5.25` | nee               | n.v.t.                     | geen stabiele npm-release                 |
| `2026.5.26` | ja                | ja                         | geneste afhankelijkheidsboom nog aanwezig |
| `2026.5.27` | ja                | ja                         | geneste afhankelijkheidsboom nog aanwezig |
| `2026.5.28` | ja                | ja                         | geneste afhankelijkheidsboom veel kleiner |

Het belangrijke onderscheid: **shrinkwrap zelf is niet het probleem**.
`v2026.5.28` wordt nog steeds met root-shrinkwrap uitgebracht. Het probleem was de pakketstructuur
waardoor npm een grote geneste OpenClaw-afhankelijkheidsboom en alle 12
`@napi-rs/canvas`-platformpakketten materialiseerde. De geneste boom is kleiner in `v2026.5.28`,
en de platformuitwaaiering van Canvas komt niet langer voor in de lokale audit.

Zie [npm-shrinkwrap](/nl/gateway/security/shrinkwrap) voor een uitleg van shrinkwrap in gewone taal en de
pakketcontroles voor maintainers.

## Interpretatie van de softwaretoeleveringsketen

Het aantal afhankelijkheden is een operationele beveiligingsmaatstaf, niet alleen een maatstaf
voor de installatiegrootte. Elk pakket vergroot de verzameling maintainers, tarballs, transitieve
updates, optionele native binaries en gedragingen tijdens de installatie die operators
moeten vertrouwen.

De richting van de opschoning is:

- houd zware en optionele mogelijkheden buiten de standaardinstallatie van de kern
- laat Plugin-pakketten hun eigen runtime-afhankelijkheidsgraaf beheren
- vermijd herstel via de pakketbeheerder tijdens het opstarten van de Gateway
- behoud deterministische installaties zonder dat native pakketten voor alle platformen
  worden gematerialiseerd
- houd installatiescripts uitgeschakeld in paden voor pakketacceptatie en metingen
- detecteer geneste afhankelijkheidsbomen en explosieve groei van optionele native afhankelijkheden vóór
  publicatie

Gerelateerde documentatie:

- [Oplossen van Plugin-afhankelijkheden](/nl/plugins/dependency-resolution)
- [Plugin-inventaris](/nl/plugins/plugin-inventory)
- [Volledige releasevalidatie](/nl/reference/full-release-validation)
