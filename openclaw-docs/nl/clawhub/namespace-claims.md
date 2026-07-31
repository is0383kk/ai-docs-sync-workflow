---
read_when:
    - Een organisatie, merk, pakketbereik, eigenaarshandle, skill-slug of pakketnaamruimte claimen
    - Een namespace oplossen die al geclaimd of gereserveerd is
    - Beslissen of je een melding, bezwaar of namespaceclaim gebruikt
sidebarTitle: Org and Namespace Claims
summary: Zo vraag je een ClawHub-beoordeling aan voor geschillen over het eigendom van een organisatie, merk, eigenaarsaccount, pakketbereik, skill-slug of naamruimte.
title: Organisatie- en naamruimteclaims
x-i18n:
    generated_at: "2026-07-27T05:26:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77a4d8090b55298c401154d116d93d4f8139d40983a45982288d8e48bcea40fb
    source_path: clawhub/namespace-claims.md
    workflow: 16
---

# Claims op organisaties en naamruimten

ClawHub gebruikt eigenaarshandles, organisatiehandles, Skills-slugs, pakketnamen van Plugins en
pakketbereiken als openbare naamruimten. Als een naamruimte bij een
bestaand project, merk, pakketecosysteem of organisatie lijkt te horen, maar al
geclaimd of gereserveerd is, misleidend is of wordt betwist op ClawHub, vraag dan het personeel deze te beoordelen
met het
[issueformulier voor claims op organisaties/naamruimten](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml).

Gebruik deze route voor openbare, niet-gevoelige beoordeling van eigendom. Gebruik geen rapporten
in het product of het bezwaarformulier voor accounts voor naamruimteclaims.

## Wanneer je een claim moet indienen

Dien een naamruimteclaim in wanneer je vindt dat het ClawHub-personeel moet beoordelen of een
naamruimte moet worden gereserveerd, overgedragen, hernoemd, verborgen, in quarantaine geplaatst, van een alias voorzien
of anderszins gewijzigd vanwege eigendom in de echte wereld.

Voorbeelden zijn:

- een organisatiehandle die overeenkomt met jouw GitHub-organisatie, project, bedrijf of community
- een pakketbereik zoals `@example-org/*` dat alleen onder de
  overeenkomende ClawHub-eigenaar mag worden gepubliceerd
- een Skills-slug of pakketnaam van een Plugin die zich lijkt voor te doen als een project
- een geschil over een merk, handelsmerk, projecthernoeming of pakketgeschiedenis
- een verwijderde, inactieve of onbereikbare eigenaar die de rechtmatige eigenaar van de naamruimte
  blokkeert

Als de vermelding onveilig, kwaadaardig of misleidend is buiten het eigendomsgeschil,
volg dan ook de relevante richtlijnen voor moderatie of beveiliging. Het formulier voor naamruimteclaims
is bedoeld voor beoordeling van eigendom, niet voor het met spoed melden van kwetsbaarheden.

## Voordat je een claim indient

Controleer eerst of je publiceert met de eigenaar die overeenkomt met de naamruimte.
Voor Plugin-pakketten moeten namen met een bereik, zoals `@example-org/example-plugin`,
worden gepubliceerd als de overeenkomende eigenaar `example-org`.

Als je de huidige eigenaar kunt beheren, herstel je de naamruimte rechtstreeks door de betreffende resource
te publiceren, hernoemen, overdragen, verbergen of verwijderen. Gebruik een claim
wanneer je de huidige eigenaar niet kunt beheren of wanneer het personeel een
geschil moet oplossen.

## Op te nemen bewijsmateriaal

Gebruik openbaar, niet-gevoelig bewijsmateriaal. Nuttig bewijs omvat:

- geschiedenis van een GitHub-organisatie, repository, release of maintainer
- officiële projectdocumentatie waarin de naamruimte wordt genoemd
- bewijs via een domein of officieel e-maildomein
- beheer van het bereik in npm, PyPI, crates.io of een ander pakketregister
- bewijs van eigendom van een handelsmerk, merk of project dat veilig
  openbaar kan worden besproken
- geschiedenis van de bronrepository, pakketgeschiedenis of openbare kennisgevingen van hernoemingen
- links naar de betwiste ClawHub-eigenaar, Skill, Plugin, het pakket of issue

Leg uit wat elke link bewijst. Het personeel moet de
relatie kunnen begrijpen zonder privéreferenties of geheimen nodig te hebben.

## Wat je niet moet opnemen

Plaats geen geheimen of privébewijs in een openbaar GitHub-issue. Neem het volgende niet op:

- API-tokens, ondertekeningssleutels of referenties
- DNS-challengetokens
- privéjuridische bestanden of contracten
- persoonlijke identiteitsdocumenten
- privé-e-mails, privébeveiligingsrapporten of vertrouwelijke klantgegevens

Het claimformulier vraagt of gevoelig bewijsmateriaal een privékanaal met het personeel
vereist. Gebruik die optie in plaats van gevoelig materiaal openbaar te plaatsen.

## Mogelijke uitkomsten

Afhankelijk van het bewijs en het risico kan het ClawHub-personeel een naamruimte reserveren,
het eigendom overdragen, een resource hernoemen, een bestaande vermelding verbergen of in quarantaine plaatsen,
een alias of omleiding toevoegen, om meer bewijs vragen of het verzoek afwijzen.

Beoordeling van een naamruimte garandeert niet dat elke overeenkomende naam wordt overgedragen.
Het personeel weegt openbaar bewijsmateriaal, bestaand gebruik, beveiligingsrisico's en gevolgen voor gebruikers af.

## Gerelateerde documentatie

- [Publiceren](/nl/clawhub/publishing)
- [Problemen oplossen](/clawhub/troubleshooting#publish-fails-because-a-namespace-is-claimed-or-reserved)
- [Moderatie en accountveiligheid](/clawhub/moderation)
- [Beveiliging](/clawhub/security)
