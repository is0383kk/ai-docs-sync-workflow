---
read_when:
    - Je hebt een duurzaam overzicht nodig van wat de Gateway heeft gedaan, zonder inhoud op te slaan
    - Je beslist of je auditing van de levenscyclus van berichten wilt inschakelen
    - Je moet uitleggen wat auditrecords wel en niet bewijzen
summary: Auditgeschiedenis met alleen metadata voor agentruns, toolacties en optionele berichtlevenscycli
title: Auditgeschiedenis
x-i18n:
    generated_at: "2026-07-27T05:45:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1005b214a674f0f888d759837bd627be458cefcf9ed61bda722499333361dc45
    source_path: gateway/audit.md
    workflow: 16
---

# Auditgeschiedenis

De Gateway houdt een begrensd auditlogboek met uitsluitend metadata bij in de gedeelde OpenClaw-statusdatabase. Het beantwoordt operationele vragen zoals "welke agent is uitgevoerd, wanneer, en hoe is de uitvoering geëindigd", "welke toolacties heeft een uitvoering verricht" en, wanneer berichtauditing is ingeschakeld, "heeft een geaccepteerd inkomend bericht de dispatch bereikt" en "heeft een uitgaand bericht een definitieve bezorgstatus bereikt".

Het logboek slaat identiteit, volgorde, herkomst, actie, status en genormaliseerde resultaatcodes op. Het slaat nooit prompts, berichtinhoud, toolargumenten, toolresultaten, bijlagen, bestandsnamen, URL's, opdrachtuitvoer of onbewerkte fouttekst op.

## Recordfamilies

Uitvoerings- en toolgebeurtenissen worden vastgelegd wanneer auditing is ingeschakeld (standaard). Gebeurtenissen in de levenscyclus van berichten zijn opt-in en standaard uitgeschakeld.

| Familie          | Acties                                                   | Standaard |
| ---------------- | -------------------------------------------------------- | --------- |
| Agentuitvoeringen | `agent.run.started`, `agent.run.finished`                | aan       |
| Toolacties       | `tool.action.started`, `tool.action.finished`            | aan       |
| Berichten        | `message.inbound.processed`, `message.outbound.finished` | uit       |

Elk record bevat een stabiele gebeurtenis-id, een monotoon oplopend logboekvolgnummer, een tijdstempel voor de levenscyclus, actor, actie, status, `schemaVersion: 1` en `redaction: "metadata_only"`. Zie [Auditrecords](/nl/cli/audit) voor het volledige overzicht van velden en queryfilters.

## Gebeurtenissen in de berichtlevenscyclus

Stel [`audit.messages`](/nl/gateway/configuration-reference#audit) in om te kiezen wat wordt vastgelegd en start vervolgens de Gateway opnieuw:

- `off` (standaard): geen berichtrecords.
- `direct`: alleen berichten in directe gesprekken.
- `all`: directe, groeps- en kanaalberichten.

Twee gezaghebbende grenzen produceren berichtrecords:

- **Inkomende** rijen worden geschreven wanneer een geaccepteerd bericht de kerndispatch bereikt, inclusief dubbele en definitieve verwerkingsresultaten.
- **Uitgaande** rijen worden geschreven wanneer gedeelde duurzame bezorging een definitief resultaat bereikt: verzonden, onderdrukt, mislukt of een expliciete `unknown` voor verzendingen met een door een crash onduidelijke status. Resultaten van wachtrijherstel en dead-letter-verwerking worden meegenomen. Elke oorspronkelijke logische antwoordpayload krijgt één definitieve rij; opsplitsing in delen en fan-out door adapters worden samengevoegd in `resultCount`.

### Classificatie van het gesprekstype

De modus `direct` vormt een privacygrens, dus een bericht wordt alleen als een direct gesprek geclassificeerd wanneer bestemmingsgegevens dit aantonen: het verzendpad heeft het gesprekstype van de bestemming opgegeven, of de routesessie voor bezorging benoemt exact het kanaal en de peer waaraan wordt bezorgd. Zwakkere signalen, zoals beleidsstatus of het oorspronkelijke gesprek, kunnen een bericht classificeren als `group` (waardoor het wordt uitgesloten van verzameling met `direct`), maar kunnen nooit `direct` claimen. Berichten waarvan niet kan worden bewezen dat ze direct zijn, worden geclassificeerd als `unknown` en niet vastgelegd in de modus `direct`. Kanalen die geen chattypen opgeven, kunnen daarom in de modus `direct` minder rijen vastleggen dan in de modus `all`.

## Privacymodel

Berichtrijen slaan nooit onbewerkte platform-id's op. Id's van accounts, gesprekken, berichten en doelen worden, wanneer correlatie beschikbaar is, alleen geëxporteerd als installatiegebonden pseudoniemen met sleutel (`hmac-sha256:v1:<keyId>:<digest>`):

- De HMAC-sleutel wordt bij het eerste gebruik gegenereerd, is per identificatortype domeingescheiden en bevindt zich in dezelfde statusdatabase als het logboek.
- Pseudoniemen zijn stabiel binnen één installatie, zodat rijen over hetzelfde gesprek kunnen worden gecorreleerd zonder de platform-id prijs te geven.
- Dit is **correlatie, geen anonimisering**: iedereen met leestoegang tot de statusdatabase beschikt ook over de sleutel en kan mogelijke onbewerkte id's met de pseudoniemen vergelijken. RPC- en CLI-exports bevatten de sleutel nooit.
- Als het sleutelmateriaal ontbreekt of beschadigd is terwijl berichtrijen worden bewaard, sluit de Gateway bij twijfel af en verwijdert deze nieuwe berichtrecords in plaats van stilzwijgend naar een nieuwe sleutel te roteren, wat de correlatie zou opsplitsen.

Uitvoerings- en toolrecords behouden `sessionKey` en `sessionId` voor correlatie; canonieke sessiesleutels kunnen zelf platformaccount- of peer-id's bevatten. Berichtrecords laten beide bewust weg.

Auditexports blijven gevoelige operationele metadata, zelfs zonder inhoud: timing, kanalen, resultaten en stabiele pseudoniemen kunnen activiteit correleren. Bescherm exports met dezelfde toegangscontroles en bewaarmethoden als andere operatorrecords.

## Dekkings- en bewijslimieten

Het logboek werkt op basis van beste inspanning en is bewust begrensd. Beschouw het als bewijs van wat is vastgelegd, niet als bewijs van wat is gebeurd:

- **Het ontbreken van een rij bewijst niets.** Inkomende berichten die vóór toelating worden verwijderd, verzendingen vanuit CLI-processen zonder actieve Gateway-recorder en Plugin-lokale of directe verzendpaden die gedeelde duurzame bezorging omzeilen, laten geen record achter.
- Schrijfacties verlopen via een begrensde achtergrondworker; een workerfout of verzadiging van de wachtrij verwijdert records en registreert één operationele waarschuwing.
- Uitgaande verzendingen met een door een crash onduidelijke status worden vastgelegd als `unknown` in plaats van dat er resultaten worden verzonnen.

Dit logboek ondersteunt foutopsporing en operationele controle. Het is geen verliesloos compliance-archief; als je dat nodig hebt, gebruik dan een extern systeem dat wordt gevoed door [OpenTelemetry](/nl/gateway/opentelemetry) of tooling op kanaalniveau.

## Opslag, bewaring en migratie

Records bevinden zich in de gedeelde statusdatabase (`state/openclaw.sqlite`) en worden buiten het kritieke bezorgpad geschreven. Query's retourneren nooit records die ouder zijn dan 30 dagen en het logboek is beperkt tot 100,000 rijen; verlopen rijen worden verwijderd tijdens het opstarten, het uurlijkse onderhoud en latere schrijfacties. Bewaaronderhoud blijft actief, zelfs wanneer verzameling is uitgeschakeld.

Bij een upgrade vanaf een Gateway met het eerdere logboek dat alleen uitvoerings- en toolrecords bevatte, wordt het schema automatisch gemigreerd tijdens het opstarten (of via `openclaw doctor --fix`); bestaande rijen en hun logboekvolgnummers blijven behouden.

## Query's uitvoeren

- CLI: [`openclaw audit`](/nl/cli/audit) met filters voor agent, sessie, uitvoering, type, status, richting, kanaal, tijdsgrenzen en cursorpaginering.
- Gateway-RPC: `audit.activity.list` (vereist `operator.read`) retourneert de geversioneerde V1-unie van activiteitsgebeurtenissen; de uitgebrachte RPC `audit.list` blijft ongewijzigd voor oudere uitvoerings-/toolclients. Zie [Gateway-protocol](/nl/gateway/protocol#audit-ledger-rpc).

## Gerelateerd

- [CLI voor auditrecords](/nl/cli/audit)
- [Configuratiereferentie](/nl/gateway/configuration-reference#audit)
- [Gateway-protocol](/nl/gateway/protocol#audit-ledger-rpc)
- [OpenTelemetry](/nl/gateway/opentelemetry)
