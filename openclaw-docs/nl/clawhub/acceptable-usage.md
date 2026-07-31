---
read_when:
    - Uploads controleren op misbruik of beleidsschendingen
    - Documentatie voor moderatie of draaiboeken voor reviewers schrijven
    - Beslissen of een skill moet worden verborgen of een gebruiker moet worden geblokkeerd
sidebarTitle: Acceptable Usage
summary: 'Marketplacebeleid: wat ClawHub toestaat en wat het niet host.'
title: Aanvaardbaar gebruik
x-i18n:
    generated_at: "2026-07-27T04:57:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ace357e7a3e9f4d242f113ad791b254e94ae8a841dd9a864a77c5bac15713132
    source_path: clawhub/acceptable-usage.md
    workflow: 16
---

# Aanvaardbaar gebruik

ClawHub host Skills, Plugins, pakketten en marketplace-metadata voor OpenClaw.
Gebruik deze pagina om te bepalen of content of publicatiegedrag op ClawHub
thuishoort.

Deze regels zijn van toepassing op wat een vermelding doet, wat gebruikers volgens
de vermelding moeten uitvoeren, hoe de vermelding zichzelf presenteert en hoe uitgevers
de functies van ClawHub voor vindbaarheid, installatie en vertrouwen gebruiken. Zie
[Moderatie en accountveiligheid](/clawhub/moderation) voor moderatiestatussen en de
accountstatus. Zie [Verzoeken over contentrechten](/clawhub/content-rights) voor claims
over auteursrecht of andere rechten.

## Toegestane content

ClawHub verwelkomt content die nuttig en begrijpelijk is en te goeder trouw wordt
gepubliceerd.

| Categorie                                        | Toegestaan wanneer                                                                                                                      |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| Productiviteit voor ontwikkelaars                | De vermelding gebruikers helpt software te bouwen, testen, migreren, debuggen, documenteren of beheren.                                 |
| Workflows voor UI, gegevens en automatisering    | Het bereik duidelijk is, vereiste inloggegevens expliciet zijn vermeld en risicovolle acties controle-, proefuitvoer-, voorbeeld- of bevestigingspaden bevatten. |
| Defensieve beveiliging, moderatie en misbruikcontrole | De tool bedoeld is voor geautoriseerde controle, bewijsmateriaal bewaart en de grenzen voor menselijke goedkeuring duidelijk houdt. |
| Persoonlijke workflows of teamworkflows          | De workflow accounts met toestemming, een transparante configuratie en expliciete machtigingen gebruikt.                               |
| Onderhouden catalogi                             | Elke vermelding onderscheidend, nuttig, nauwkeurig beschreven en redelijkerwijs onderhouden is.                                        |

De context is belangrijk. Hetzelfde onderwerp kan aanvaardbaar zijn in een beperkte
defensieve omgeving of een omgeving op basis van toestemming, maar onaanvaardbaar
wanneer het als workflow voor misbruik wordt aangeboden.

## Niet-toegestane content

ClawHub host geen content waarvan het hoofddoel misbruik, misleiding, onveilige
uitvoering of schending van rechten is.

| Categorie                                                    | Niet toegestaan                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ongeautoriseerde toegang of omzeiling van beveiliging       | Omzeiling van authenticatie, overname van accounts, misbruik van frequentielimieten, overname van livegesprekken of agents, diefstal van herbruikbare sessies of automatische goedkeuring van koppelingsprocessen voor niet-goedgekeurde gebruikers. |
| Platformmisbruik en omzeiling van blokkades                  | Verborgen accounts na blokkades, accounts opwarmen of farmen, nepinteractie, automatisering van meerdere accounts, massaal publiceren, spambots of automatisering die is gebouwd om detectie te vermijden. |
| Fraude, oplichting en misleidende financiële workflows      | Valse certificaten of facturen, misleidende betalingsprocessen, frauduleuze benadering, vals sociaal bewijs, workflows met synthetische identiteiten voor fraude of tools voor uitgaven/afschrijvingen zonder duidelijke menselijke goedkeuring. |
| Privacy-invasieve gegevensverrijking of surveillance        | Contactgegevens scrapen voor spam, doxing, stalking, het verzamelen van leads in combinatie met ongevraagde benadering, heimelijke monitoring, biometrische matching zonder toestemming of het gebruik van gelekte gegevens of dumps van datalekken. |
| Imitatie of identiteitsmanipulatie zonder toestemming       | Gezichtsverwisseling, digitale tweelingen, gekloonde influencers, neppersona's of andere tools die worden gebruikt om zich als iemand anders voor te doen of mensen te misleiden. |
| Expliciete seksuele content of het genereren van content voor volwassenen zonder veiligheidsmaatregelen | Het genereren van NSFW-afbeeldingen, -video's of -content; wrappers voor content voor volwassenen rond API's van derden; of vermeldingen waarvan expliciete seksuele content het hoofddoel is. |
| Verborgen, onveilige of misleidende uitvoeringsvereisten    | Verdoezelde installatieopdrachten, pipe-to-shell-installatieprogramma's zoals gedownloade content die met `sh` of `bash` wordt uitgevoerd zonder dat duidelijke controle mogelijk is, niet-aangegeven vereisten voor geheimen of privésleutels, externe uitvoering van `npx @latest` zonder dat duidelijke controle mogelijk is, of metadata die verbergen wat er werkelijk nodig is om de vermelding uit te voeren. |
| Materiaal dat auteursrechten of andere rechten schendt      | Skills, Plugins, documentatie, merkmateriaal of propriëtaire code van iemand anders zonder toestemming opnieuw publiceren; licentievoorwaarden schenden; of zich voordoen als de oorspronkelijke auteur of uitgever. |

## Niet-toegestaan gedrag op de marketplace

ClawHub controleert ook hoe uitgevers de marketplace gebruiken. Gebruik ClawHub niet
om vindbaarheid, statistieken, vertrouwenssignalen, moderatiesystemen of de aandacht
van gebruikers te manipuleren.

Niet-toegestaan gedrag op de marketplace omvat:

- grote aantallen vermeldingen met weinig inspanning, duplicaten, tijdelijke aanduidingen of
  automatisch gegenereerde vermeldingen in bulk publiceren die geen echte waarde voor gebruikers lijken te hebben
- zoek- of categoriepagina's overspoelen met vrijwel identieke Skills of Plugins
- honderden vermeldingen publiceren met weinig of geen gebruik, onderhoud, duidelijkheid over de bron
  of betekenisvol onderscheid
- installaties, downloads, sterren of andere interactiestatistieken kunstmatig
  verhogen via automatisering, lussen voor zelfinstallatie, nepaccounts, gecoördineerde
  activiteit, betaalde interactie of ander niet-organisch gedrag
- accounts maken of rouleren om moderatie, blokkades, beperkingen voor uitgevers of
  beoordeling door de marketplace te omzeilen
- gebruikers misleiden over eigendom, bron, mogelijkheden, beveiligingsniveau,
  installatievereisten of verbondenheid met een ander project of een andere uitgever
- herhaaldelijk content uploaden die al is verborgen, verwijderd of geblokkeerd
  zonder het onderliggende probleem op te lossen

Publicatie in grote volumes is niet automatisch misbruik. Grote catalogi zijn
aanvaardbaar wanneer de vermeldingen wezenlijk van elkaar verschillen, nauwkeurig
zijn beschreven, worden onderhouden en door echte gebruikers worden gebruikt. Grote
catalogi worden een probleem voor vertrouwen en veiligheid wanneer het volume gepaard
gaat met oppervlakkige, dubbele, misleidende, niet-onderhouden of kunstmatig
gepromote vermeldingen.

## Contentrechten

Als je denkt dat content op ClawHub inbreuk maakt op jouw auteursrecht of andere
rechten, gebruik je [Verzoeken over contentrechten](/clawhub/content-rights). Gebruik
normale meldingen op de marketplace niet voor claims over auteursrecht of andere
rechten, tenzij de vermelding ook onveilig, schadelijk of misleidend is.

## Controle en handhaving

ClawHub kan geautomatiseerde controles, statistische signalen van misbruik,
gebruikersmeldingen en controles door medewerkers gebruiken om onveilige content of
misbruik bij het publiceren te identificeren. Een signaal bewijst op zichzelf geen
misbruik; het helpt ClawHub te bepalen wat moet worden gecontroleerd.

We kunnen:

- vermeldingen die de regels overtreden verbergen, vasthouden, verwijderen, voorlopig verwijderen
  of, wanneer dit voor het resourcetype wordt ondersteund, definitief verwijderen
- downloads of installaties van onveilige releases blokkeren
- API-tokens intrekken
- bijbehorende content voorlopig verwijderen
- publicatietoegang beperken
- herhaaldelijke of ernstige overtreders blokkeren

We garanderen niet dat bij duidelijk misbruik eerst een waarschuwing wordt gegeven.
Zie [Moderatie en accountveiligheid](/clawhub/moderation) voor meldingen,
moderatieblokkades, verborgen vermeldingen, blokkades en de accountstatus.
