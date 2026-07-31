---
read_when:
    - Opvolging na feedback van Barnacle of ClawSweeper
    - ClawSweeper om een review vragen
    - Barnacle, ClawSweeper, verouderde labels of automatische sluitingen debuggen
sidebarTitle: PR review flow
summary: Hoe feedback van Barnacle en ClawSweeper helpt om pull requests voor OpenClaw door het reviewproces te loodsen.
title: Reviewflow voor pull requests
x-i18n:
    generated_at: "2026-07-27T05:15:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9bec4578d55d2279450e991480467946db7da5ca956f85c35b4221190b2babe
    source_path: reference/pull-request-review-flow.md
    workflow: 16
---

Deze pagina legt de reviewflow uit nadat je een pull request voor OpenClaw hebt geopend of bijgewerkt: wat Barnacle en ClawSweeper doen, hoe je de PR verbetert op basis van hun feedback en wat je moet controleren wanneer de automatisering stil blijft.

Barnacle en ClawSweeper helpen beheerders de reviewwachtrij werkbaar te houden. Ze vervangen het oordeel van beheerders niet.

## Barnacle

Barnacle verzorgt deterministische GitHub-triage. Het zoekt naar bekende gevallen voor wachtrijbeheer en reageert met labels, opmerkingen of sluitingen.

Barnacle kan actie ondernemen wanneer:

- een PR-beschrijving grotendeels leeg is of probleemcontext ontbreekt;
- een PR geen bruikbaar bewijs bevat;
- een wijziging die alleen documentatie, tests, refactoring, CI of infrastructuur betreft geen gekoppelde beheerderscontext bevat;
- een wijziging in ClawHub of een plugin lijkt thuis te horen in plaats van in de kern;
- een branch niet-gerelateerd werk bevat;
- een auteur meer dan 20 openstaande PR's heeft.

Barnacle wordt uitgevoerd vanuit vertrouwde workflowcode van de repository. Het checkt code van bijdragers niet uit en voert deze niet uit.

De meeste routeringslabels zijn signalen voor beheerders of automatisering, dus bijdragers hoeven zelf geen labels toe te voegen.

## ClawSweeper

ClawSweeper is de AI-ondersteunde bot voor reviews en onderhoud van OpenClaw-repository's. De bot kan PR's beoordelen, bewijs evalueren, blijvende reviewopmerkingen plaatsen en beheerders helpen met beveiligde herstel- of automatische samenvoegflows.

Een positief resultaat van ClawSweeper is ondersteunend bewijs, geen goedkeuring door een beheerder. Beheerders beslissen nog steeds of en wanneer een PR gereed is om te worden samengevoegd.

ClawSweeper werkt met een wachtrij. Verwacht geen onmiddellijke reactie na het openen van een PR, het pushen van een commit of het toevoegen van een reviewverzoek. Ook labelupdates na een ClawSweeper-run kunnen enige tijd duren.

Nieuwe PR's komen in de reviewwachtrij van ClawSweeper terecht. Beheerders kunnen ook review-, herstel- of automatische samenvoegflows in de wachtrij plaatsen met labels of opdrachten. Vraag bij gewone updates van bijdragers pas om een nieuwe review door ClawSweeper nadat je de branch, PR-beschrijving, het bewijs of de code hebt bijgewerkt. Vraag vervolgens met een nieuwe PR-opmerking om een nieuwe review:

```text
@clawsweeper re-review
```

PR-auteurs kunnen ook `@clawsweeper re-run` gebruiken; gebruikers met schrijftoegang tot de repository kunnen beide opdrachten voor elk openstaand item gebruiken. De gewone opdracht `@clawsweeper review` is alleen voor beheerders. Wees geduldig: opnieuw vragen voordat de gevraagde wijzigingen aanwezig zijn, veroorzaakt alleen maar ruis in de wachtrij.

Wanneer ClawSweeper reviewgesprekken achterlaat, behandel je die als normale reviewfeedback en gebruik je de onderstaande vervolgchecklist.

Als een menselijke bijdrager of beheerder de PR heeft overgenomen en er actief aan werkt, roep ClawSweeper dan niet aan en werk niet tegelijkertijd op een andere manier aan de PR. Laat de menselijke review of het herstel eerst afronden. Als de activiteit stopt, controleer dan of de auteur is gevraagd bewijs te leveren of andere updates uit te voeren.

## Een PR tijdens de review verbeteren

Zodra Barnacle, ClawSweeper of een beheerder reageert, gebruik je die feedback als checklist voor de volgende stappen voor de PR.

1. Lees de `Rank-up moves:` en `Proof guidance:` van ClawSweeper als de actielijst voor die PR. Beoordelingen en labels zijn reviewsignalen, geen vaste samenvoegdoelen.
2. Push de gevraagde wijziging in de code of documentatie en werk de PR-beschrijving bij wanneer het probleem, de oplossing, de gevolgen voor gebruikers of het bewijs is gewijzigd.
3. Voeg het gevraagde bewijs toe en gebruik bewijs dat bij de wijziging past.
4. Los afgehandelde reviewgesprekken zelf op. Reageer en laat een gesprek alleen open wanneer je een oordeel van een beheerder of reviewer nodig hebt.
5. Vraag pas om een nieuwe review wanneer de branch, PR-beschrijving, het bewijs en de relevante CI-resultaten actueel zijn. Meerdere update- en reviewcycli tussen de auteur, beheerder en ClawSweeper zijn normaal.
6. Houd de discussie waar mogelijk bij de PR. Ga alleen naar `#clawtributors` op Discord wanneer voor de PR afstemming met beheerders nodig is, de automatisering geblokkeerd lijkt of de volgende beslissing moeilijk via GitHub-opmerkingen kan worden genomen. Voeg de PR-link, de huidige status en de specifieke vraag of het resterende bewijs toe.

Houd de PR-beschrijving actueel. Opmerkingen helpen bij de discussie, maar de PR-beschrijving is de blijvende samenvatting die beheerders en automatisering opnieuw bekijken.

`status: ⏳ waiting on author` betekent dat de volgende actie bij de PR-auteur ligt: werk de branch, PR-beschrijving of het bewijs bij, of reageer met de ontbrekende context voordat je om een nieuwe review vraagt.

Bruikbaar bewijs omvat gerichte testuitvoer, CI-resultaten, schermafbeeldingen, opnamen, terminaluitvoer, livewaarnemingen, geredigeerde logboeken of links naar artefacten. Voeg voor visuele wijzigingen waar mogelijk schermafbeeldingen van vóór en na de wijziging toe. Geef voor bewijsbestanden de voorkeur aan links naar CI-artefacten, naar GitHub geüploade schermafbeeldingen of opnamen, of een kort geredigeerd logboekfragment. Commit geen gegenereerde bewijsbestanden, tenzij deze deel uitmaken van de daadwerkelijke wijziging in de documentatie, tests of het product.

Het redigeren van gevoelige gegevens is de verantwoordelijkheid van de bijdrager. Verwijder geheimen, tokens, privé-URL's, gebruikersgegevens en niet-gerelateerde logboeken voordat je bewijs plaatst.

OpenClaw gebruikt ook afzonderlijke automatisering voor verouderde items. Niet-toegewezen issues en PR's kunnen na 14 dagen zonder activiteit als verouderd worden gemarkeerd en vervolgens na nog eens 7 dagen zonder activiteit worden gesloten. Toegewezen PR's worden 27 dagen na opening als verouderd gemarkeerd, ongeacht latere updates, en vervolgens na 7 dagen zonder activiteit sinds de verouderingsmarkering gesloten. Als een toegewezen PR nog actief is, stem dan af met de beheerder die eraan werkt.

## Wanneer de automatisering stil blijft

De automatisering kan stil blijven wanneer een beheerder het item al behandelt, een review- of herstelverzoek nog in de wachtrij staat, de gebeurtenis routinematig is of de ClawSweeper-route niet voor de gevraagde actie is geconfigureerd.

De automatisering kan ook afzien van actie wanneer een vertrouwde workflow niet-vertrouwde code van bijdragers zou moeten uitvoeren. In dat geval gebruiken beheerders in plaats daarvan een normale review of een veiligere workflow.

## Problemen oplossen

Als ClawSweeper niet onmiddellijk reageert, wacht dan voordat je het opnieuw probeert. De service werkt met een wachtrij en herhaalde opmerkingen of labelwijzigingen kunnen de thread moeilijker te beoordelen maken zonder de wachtrij te versnellen.

Controleer voordat je om hulp vraagt of:

- de PR-beschrijving actueel is;
- de nieuwste commit de gevraagde wijziging bevat;
- CI is voltooid, of in de PR-beschrijving wordt uitgelegd waarom een resterende fout geen verband houdt met de PR;
- het nieuwste reviewverzoek als PR-opmerking is geplaatst:
  `@clawsweeper re-review`;
- een beheerder of bijdrager niet al actief aan de PR werkt;
- het nieuwste verzoek niet nog binnen de normale wachttijd van de ClawSweeper-wachtrij valt.

Als ClawSweeper nog steeds niet heeft gereageerd enkele uren nadat de PR actueel is geworden, of als de PR door automatisering geblokkeerd lijkt, vraag dan om hulp in `#clawtributors` op Discord. Voeg de PR-link toe, wat je verwachtte, wanneer je het vroeg en wat er sinds de laatste botopmerking is gewijzigd.

## De automatisering forken

Projecten die vergelijkbare reviewautomatisering willen, kunnen ClawSweeper bestuderen of forken:

- [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)
- [ClawSweeper-documentatie](https://clawsweeper.bot/)

## Gerelateerd

- [Bijdragen](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [CI-pijplijn](/nl/ci)
