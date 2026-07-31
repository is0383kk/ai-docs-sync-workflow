---
read_when:
    - Fouten met ontbrekende operatorbereiken opsporen
    - Goedkeuringen voor het koppelen van apparaten of nodes beoordelen
    - Gateway-RPC-methoden toevoegen of classificeren
summary: Operatorrollen, bereiken en controles op het moment van goedkeuring voor Gateway-clients
title: Operatorbereiken
x-i18n:
    generated_at: "2026-07-27T05:12:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

Operatorbereiken bepalen wat een Gateway-client kan doen nadat deze zich heeft geauthenticeerd.
Ze vormen een beveiligingsmechanisme voor het besturingsvlak binnen één vertrouwd Gateway-operatordomein,
geen isolatie tegen vijandige tenants. Voor een sterke scheiding tussen personen,
teams of machines voer je afzonderlijke Gateways uit onder afzonderlijke OS-gebruikers of hosts.

Gerelateerd: [Beveiliging](/nl/gateway/security), [Gateway-protocol](/nl/gateway/protocol),
[Gateway-koppeling](/nl/gateway/pairing), [Apparaten-CLI](/nl/cli/devices).

## Rollen

Elke Gateway WebSocket-client maakt verbinding met één rol:

- `operator`: clients voor het besturingsvlak, zoals CLI, Control UI, automatisering en
  vertrouwde hulpprocessen.
- `node`: capaciteitshosts (macOS, iOS, Android, headless) die
  opdrachten beschikbaar stellen via `node.invoke`.

RPC-methoden voor operators vereisen de rol `operator`; methoden die afkomstig zijn van nodes
vereisen de rol `node`.

## Bereikniveaus

| Bereik                  | Betekenis                                                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | Alleen-lezenstatus, lijsten, catalogus, logboeken, sessieleesbewerkingen en andere niet-wijzigende aanroepen.                                                 |
| `operator.write`        | Wijzigende operatoracties: berichten verzenden, tools aanroepen, instellingen voor spraak en stem bijwerken, doorsturen van node-opdrachten. Voldoet ook aan `operator.read`. |
| `operator.admin`        | Beheerderstoegang. Voldoet aan elk `operator.*`-bereik. Vereist voor configuratiewijzigingen, updates, native hooks, gereserveerde naamruimten en goedkeuringen met een hoog risico. |
| `operator.pairing`      | Beheer van apparaat- en nodekoppelingen: weergeven, goedkeuren, afwijzen, verwijderen, roteren, intrekken.                                                     |
| `operator.approvals`    | API's voor exec- en plugingoedkeuringen.                                                                                                                       |
| `operator.questions`    | Interactieve vragen weergeven, lezen, beantwoorden en afhandelen.                                                                                              |
| `operator.talk.secrets` | Talk-configuratie lezen, inclusief geheimen.                                                                                                                    |

Onbekende toekomstige `operator.*`-bereiken vereisen een exacte overeenkomst, tenzij de aanroeper
al `operator.admin` heeft.

## Methodebereik is slechts de eerste controle

Elke Gateway-RPC heeft een methodebereik met minimale bevoegdheden dat bepaalt of een
verzoek de handler bereikt. Methoden die rekening houden met parameters leiden dat bereik vóór
de dispatch af, zodat autorisatiefouten één canoniek gestructureerd antwoord hebben:

- `agent` vereist `operator.write` voor gewone beurten en `operator.admin` voor
  sessielevenscyclusopdrachten `/new` of `/reset`.
- `node.invoke` vereist `operator.write` voor gewone doorstuuropdrachten en
  `operator.admin` voor `browser.proxy`, `fs.listDir` en `terminal.upload`.
- `talk.config` vereist `operator.read`; `includeSecrets: true` vereist ook
  `operator.talk.secrets`.

Sommige handlers voeren vervolgens strengere controles uit op basis van het concrete object dat
wordt goedgekeurd of gewijzigd:

- `device.pair.approve` is bereikbaar met `operator.pairing`, maar bij het goedkeuren van een
  operatorapparaat kunnen alleen bereiken worden uitgegeven of behouden die de aanroeper al heeft.
- `node.pair.approve` is bereikbaar met `operator.pairing` en leidt vervolgens aanvullende
  goedkeuringsbereiken af uit de opgegeven opdrachtenlijst van de wachtende node.
- `chat.send` is een methode met schrijfbereik, maar de chatopdrachten
  `/config set` en `/config unset` vereisen daarnaast `operator.admin`,
  ongeacht het bereik van de aanroeper voor het verzenden van chatberichten.

Hierdoor kunnen operators met een beperkter bereik koppelingsacties met een laag risico uitvoeren
zonder dat alle koppelingsgoedkeuringen uitsluitend voor beheerders toegankelijk worden.

RPC's voor sessiewijzigingen worden geautoriseerd op basis van hun overeengekomen operatorbereiken,
onafhankelijk van de `client.id` of `client.mode` van de client die verbinding maakt. De
clientidentiteit kan nog steeds invloed hebben op het beleid voor verbindingen en apparaatauthenticatie, maar
verleent noch ontneemt bevoegdheid voor sessiewijzigingen.

## Goedkeuringen voor apparaatkoppelingen

Apparaatkoppelingsrecords zijn de duurzame bron van goedgekeurde rollen en bereiken.
Een reeds gekoppeld apparaat krijgt niet stilzwijgend ruimere toegang: een nieuwe verbinding
die om een ruimere rol of ruimere bereiken vraagt, maakt een nieuw wachtend upgradeverzoek aan.

Een apparaatverzoek goedkeuren:

- Een verzoek zonder operatorrol vereist geen goedkeuring voor een operatorbereik.
- Een verzoek voor een niet-operatorapparaatrol (bijvoorbeeld `node`) vereist
  `operator.admin`, hoewel `device.pair.approve` zelf alleen
  `operator.pairing` vereist.
- Een verzoek voor `operator.read`, `operator.write`, `operator.approvals`,
  `operator.questions`, `operator.pairing` of `operator.talk.secrets` vereist
  dat de aanroeper dat bereik, of `operator.admin`, al heeft.
- Een verzoek voor `operator.admin` vereist `operator.admin`.
- Een herstelverzoek zonder expliciete bereiken kan de bereiken van het bestaande
  operatortoken overnemen; als dat token een beheerdersbereik heeft, vereist goedkeuring nog steeds
  `operator.admin`.

Sessies met een gedeeld geheim zonder beheerdersrechten en sessies met een vertrouwde proxy kunnen
verzoeken voor operatorapparaten alleen goedkeuren binnen hun eigen opgegeven operatorbereiken; het goedkeuren
van niet-operatorrollen is uitsluitend voor beheerders, zelfs wanneer die sessies anders
`operator.pairing` kunnen gebruiken.

Voor tokensessies van gekoppelde apparaten is het beheer beperkt tot het eigen apparaat, tenzij de aanroeper
`operator.admin` heeft: een aanroeper zonder beheerdersrechten ziet alleen de eigen koppelingsvermeldingen en
kan alleen de eigen apparaatvermelding goedkeuren, afwijzen, roteren, intrekken of verwijderen.

## Goedkeuringen voor nodekoppelingen

Verouderde `node.pair.*`-methoden gebruiken een afzonderlijk door de Gateway beheerd opslagmedium voor nodekoppelingen.
WS-nodes gebruiken in plaats daarvan apparaatkoppeling (`role: node`), maar dezelfde
goedkeuringsterminologie is van toepassing. Zie [Gateway-koppeling](/nl/gateway/pairing) voor de relatie tussen de twee
opslagmedia.

`node.pair.approve` leidt aanvullende vereiste bereiken af uit de
opdrachtenlijst van het wachtende verzoek:

| Opgegeven opdrachten                                                                                                 | Vereiste bereiken                      |
| -------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| geen                                                                                                                 | `operator.pairing`                     |
| gewone node-opdrachten                                                                                               | `operator.pairing` + `operator.write` |
| `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` of `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

Het goedkeuren van een nodeverklaring schakelt geen opdrachten in waarvoor een afzonderlijke
runtime-toestaanlijst geldt. Het goedkeuren van een node die
`computer.act` opgeeft, vereist bijvoorbeeld een koppelings- plus schrijfbereik, maar registreert alleen het oppervlak.
Een beheerder of eigenaar moet `computer.act` nog steeds activeren. Zolang deze functie
geactiveerd blijft, vereist het aanroepen ervan via `node.invoke` een schrijfbereik, maar niet voor elke
actie een beheerdersbereik.

Nodekoppeling legt identiteit en vertrouwen vast; het vervangt niet het eigen
`system.run`-beleid voor exec-goedkeuringen van een node.

## Authenticatie met gedeeld geheim

Authenticatie met een gedeeld Gateway-token/wachtwoord wordt behandeld als vertrouwde operatortoegang voor
die Gateway. OpenAI-compatibele HTTP-oppervlakken, `/tools/invoke` en HTTP-eindpunten
voor sessiegeschiedenis herstellen de volledige standaardset operatorbereiken voor
bearer-authenticatie met een gedeeld geheim, zelfs als een aanroeper beperktere opgegeven bereiken verzendt.

Modi die een identiteit bevatten, zoals authenticatie via een vertrouwde proxy of `none` met privé-ingang,
kunnen expliciet opgegeven bereiken nog steeds respecteren. Gebruik afzonderlijke Gateways voor een echte scheiding
van vertrouwensgrenzen.
