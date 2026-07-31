---
read_when:
    - Een skill, plugin of pakket melden
    - Herstellen van een vermelding die is vastgehouden, verborgen of geblokkeerd
    - Inzicht in ClawHub-moderatie, verbanningen en accountstatus
sidebarTitle: Moderation and Account Safety
summary: Hoe meldingen, moderatieblokkeringen, verborgen vermeldingen, verbanningen en accountstatus werken in ClawHub.
title: Moderatie en accountveiligheid
x-i18n:
    generated_at: "2026-07-27T05:03:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54c1e0860411e6599923ef4d7db65d5cd5406ec63bf67c52968b4f99d893ffef
    source_path: clawhub/moderation.md
    workflow: 16
---

# Moderatie en accountveiligheid

ClawHub staat open voor publicaties, maar openbare vindbaarheid en installatiekanalen hebben nog steeds
waarborgen nodig. Meldingen, moderatieblokkeringen, verborgen vermeldingen en accountmaatregelen
helpen gebruikers te beschermen wanneer een release of account onveilig, misleidend of niet
conform het beleid lijkt.

Deze pagina behandelt moderatie en accountstatus. Voor auditlabels zoals
`Pass`, `Review`, `Warn`, `Malicious` en risiconiveau, zie
[Beveiligingsaudits](/clawhub/security-audits).

Zie ook [Beveiliging](/clawhub/security) en
[Aanvaardbaar gebruik](/clawhub/acceptable-usage). Gebruik voor zorgen over auteursrecht of andere
inhoudsrechten [Verzoeken over inhoudsrechten](/clawhub/content-rights).

## Meldingen

Ingelogde gebruikers kunnen Skills, plugins en pakketten melden.

Gebruik ClawHub-meldingen alleen voor onveilige marktplaatsinhoud, zoals:

- schadelijke vermeldingen
- misleidende metadata
- niet-aangegeven vereisten voor inloggegevens of machtigingen
- verdachte installatie-instructies
- identiteitsmisbruik
- registraties te kwader trouw of misbruik van handelsmerken
- inhoud die [Aanvaardbaar gebruik](/clawhub/acceptable-usage) schendt

Gebruik de knop **Skill melden** op een Skill-pagina, of de opdracht/API voor
het melden van pakketten.

Gebruik ClawHub-meldingen niet voor kwetsbaarheden in de eigen broncode van een
Skill of Plugin van derden. Meld deze rechtstreeks bij de uitgever of de
broncoderepository waarnaar vanuit de vermelding wordt verwezen. ClawHub onderhoudt of
patcht geen code van Skills of plugins van derden.

GitHub Security Advisories voor `openclaw/clawhub` zijn bedoeld voor kwetsbaarheden in
ClawHub zelf. Voorbeelden zijn fouten in de website, API, CLI, het register, de authenticatie,
scans, moderatie of de vertrouwensgrenzen voor downloaden/installeren. Gebruik ClawHub-
advisories niet voor kwetsbaarheden in Skills of plugins van derden.

Goede meldingen zijn specifiek en bruikbaar. Misbruik van de meldfunctie kan zelf leiden tot
accountmaatregelen.

## Claims op organisaties en naamruimten

Geschillen over eigendom van een organisatie, merk, pakketbereik, eigenaarsnaam of naamruimte moeten
via het proces [Claims op organisaties en naamruimten](/clawhub/namespace-claims) worden behandeld, niet via
de meldfunctie in het product of het bezwaarformulier voor accounts.

Gebruik dat proces wanneer ClawHub-medewerkers niet-gevoelig bewijs moeten beoordelen dat een
naamruimte moet worden gereserveerd, overgedragen, hernoemd, verborgen, in quarantaine geplaatst, van een alias voorzien
of anderszins beoordeeld. Neem geen geheimen, privédocumenten, vertrouwelijke juridische
bestanden, persoonlijke identiteitsdocumenten, API-tokens of DNS-challengetokens op in een
openbare issue.

## Moderatieblokkeringen

Sommige ernstige bevindingen of beleidsproblemen kunnen ertoe leiden dat een uitgever of vermelding onder een
moderatieblokkering wordt geplaatst. Wanneer dit gebeurt, kan de betreffende inhoud worden verborgen voor openbare
vindbaarheid, of kunnen toekomstige publicaties aanvankelijk verborgen zijn totdat het probleem is beoordeeld.

Moderatieblokkeringen zijn bedoeld om gebruikers te beschermen terwijl ClawHub gevallen met een hoog risico
afhandelt. Ze kunnen ook worden opgeheven wanneer een fout-positief resultaat is bevestigd.

## Verborgen of geblokkeerde vermeldingen

Een vermelding kan worden vastgehouden, verborgen, in quarantaine geplaatst, ingetrokken of anderszins niet beschikbaar zijn op
openbare installatiekanalen.

Als je een van deze statussen ziet, installeer de release dan niet, tenzij de eigenaar
het probleem oplost of de moderatie de vermelding herstelt.

Eigenaren kunnen nog steeds diagnostische gegevens zien voor hun eigen vastgehouden of verborgen vermeldingen. Deze
diagnostische gegevens helpen uit te leggen wat er is gebeurd en wat er moet veranderen voordat de
vermelding kan terugkeren naar openbare kanalen.

## Blokkeringen en accountstatus

Accounts die het ClawHub-beleid schenden, kunnen hun publicatietoegang verliezen. Ernstig misbruik kan
leiden tot accountblokkeringen, intrekking van tokens, verborgen inhoud of verwijderde vermeldingen.
Signalen voor misbruikdruk door uitgevers worden dagelijks gecontroleerd. Signalen die
de drempel voor een mogelijke blokkering van ClawHub bereiken, kunnen een automatische waarschuwing activeren. Als de volgende
in aanmerking komende scan na de waarschuwingstermijn de uitgever nog steeds binnen de
drempel voor een mogelijke blokkering plaatst, kan ClawHub de accountmaatregel automatisch toepassen.
Beoordelingssignalen met een lagere betrouwbaarheid en een begrensd tijdsvenster vallen buiten automatische
handhaving.

Verwijderde, geblokkeerde of uitgeschakelde accounts kunnen geen ClawHub API-tokens gebruiken. Als CLI-authenticatie
na een accountmaatregel niet meer werkt, meld je dan aan bij de webinterface om de accountstatus
te bekijken. Als aanmelden of normale CLI-toegang wordt geblokkeerd door een blokkering of uitgeschakeld account,
gebruik dan het [ClawHub-bezwaarformulier](https://appeals.openclaw.ai/) voor een herstelbeoordeling.

Als een door een scanner geactiveerde e-mail een versie van een Skill of Plugin als schadelijk aanduidt,
download dan de opgeslagen scanresultaten voor de geblokkeerde ingediende versie:
`clawhub scan download <slug> --version <version>`. Voeg voor plugins
`--kind plugin` toe. Bekijk de scanuitvoer, corrigeer de vermelding, verhoog het versienummer
en upload de gecorrigeerde versie.

## Richtlijnen voor uitgevers

Om fout-positieve resultaten te verminderen en het vertrouwen van gebruikers te verbeteren:

- houd namen, samenvattingen, tags en wijzigingslogboeken correct
- vermeld vereiste omgevingsvariabelen en machtigingen
- vermijd versluierde installatieopdrachten
- verwijs waar mogelijk naar de broncode
- gebruik proefuitvoeringen voordat je plugins publiceert
- reageer duidelijk als gebruikers of moderators vragen stellen over het gedrag van een release
