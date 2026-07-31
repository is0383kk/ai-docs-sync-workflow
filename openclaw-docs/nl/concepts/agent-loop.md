---
read_when:
    - Je hebt een exact stappenplan van de agentlus of levenscyclusgebeurtenissen nodig
    - Je wijzigt het wachtrijbeheer van sessies, het schrijven van transcripten of het gedrag van schrijfvergrendelingen voor sessies
summary: Levenscyclus van de agentloop, streams en wachtsemantiek
title: Agentlus
x-i18n:
    generated_at: "2026-07-27T05:29:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1d0102ffb6ebf572ea0201470db138775be33b0f0b655d9d08742177be5f3f31
    source_path: concepts/agent-loop.md
    workflow: 16
---

De agentlus is de geserialiseerde uitvoering per sessie die een bericht omzet in
acties en een antwoord: ontvangst, contextopbouw, modelinferentie, uitvoering van
tools, streaming, persistentie.

## Ingangspunten

- Gateway-RPC: `agent` en `agent.wait`.
- CLI: `openclaw agent`.

## Uitvoeringsvolgorde

1. `agent` RPC valideert parameters, bepaalt de sessie (`sessionKey`/`sessionId`), slaat sessiemetadata persistent op en retourneert onmiddellijk `{ runId, acceptedAt }`.
2. `agentCommand` voert de beurt uit: bepaalt de standaardwaarden voor model + denken/uitgebreid/trace, laadt de Skills-snapshot, roept `runEmbeddedAgent` aan en verzendt een terugvalgebeurtenis **einde/fout van levenscyclus** als de ingebedde lus er nog geen heeft verzonden.
3. `runEmbeddedAgent`: serialiseert uitvoeringen via wachtrijen per sessie en globale wachtrijen, bepaalt het model + authenticatieprofiel, bouwt de OpenClaw-sessie, abonneert zich op runtimegebeurtenissen, streamt assistent-/tooldelta's, handhaaft de uitvoeringstime-out (met afbreking bij het verstrijken ervan) en retourneert payloads plus gebruiksmetadata. Voor beurten van de Codex-appserver breekt dit ook een geaccepteerde beurt af die vóór een terminale gebeurtenis geen appservervoortgang meer produceert.
4. `subscribeEmbeddedAgentSession` koppelt runtimegebeurtenissen aan de `agent`-stream: toolgebeurtenissen aan `stream: "tool"`, assistentdelta's aan `stream: "assistant"`, levenscyclusgebeurtenissen aan `stream: "lifecycle"` (`phase: "start" | "end" | "error"`).
5. `agent.wait` (`waitForAgentRun`) wacht op **einde/fout van levenscyclus** op een `runId` en retourneert `{ status: ok|error|timeout, startedAt, endedAt, error? }`.

## Wachtrijen en gelijktijdigheid

Uitvoeringen worden per sessiesleutel (sessielane) geserialiseerd en optioneel via een globale lane geleid, waardoor conflicten tussen tools en sessies worden voorkomen. Berichtenkanalen kiezen een wachtrijmodus (steer/followup/collect/interrupt) die dit lanesysteem voedt; zie [Opdrachtwachtrij](/nl/concepts/queue).

Het schrijven van transcripten wordt bovendien beschermd door een sessieschrijfvergrendeling op het sessiebestand. De vergrendeling is procesbewust en bestandsgebaseerd, zodat deze schrijvers detecteert die de wachtrij binnen het proces omzeilen of uit een ander proces komen. Schrijvers wachten standaard maximaal 60 seconden (omgevingsoverride `OPENCLAW_SESSION_WRITE_LOCK_ACQUIRE_TIMEOUT_MS`) voordat ze melden dat de sessie bezet is.

Sessieschrijfvergrendelingen zijn standaard niet herintredend. Een helper die opzettelijk het verkrijgen van dezelfde vergrendeling nestelt en daarbij één logische schrijver behoudt, moet dit inschakelen met `allowReentrant: true`.

## Voorbereiding van sessie en werkruimte

- De werkruimte wordt bepaald en aangemaakt; uitvoeringen in een sandbox kunnen worden omgeleid naar de hoofdmap van een sandboxwerkruimte.
- Skills worden geladen (of hergebruikt vanuit een snapshot) en in de omgeving en prompt geïnjecteerd.
- Bootstrap-/contextbestanden worden bepaald en in de systeemprompt geïnjecteerd.
- Er wordt een sessieschrijfvergrendeling verkregen en het doel voor het sessietranscript wordt voorbereid voordat het streamen begint. Elk later pad voor het herschrijven, comprimeren of inkorten van het transcript moet dezelfde vergrendeling verkrijgen voordat de SQLite-transcriptrijen worden gewijzigd.

## Promptopbouw

De systeemprompt wordt opgebouwd uit de basisprompt van OpenClaw, de Skills-prompt, bootstrapcontext en overrides per uitvoering. Modelspecifieke limieten en gereserveerde tokens voor Compaction worden gehandhaafd. Zie [Systeemprompt](/nl/concepts/system-prompt) voor wat het model ziet.

## Hooks

OpenClaw heeft twee hooksystemen:

- **Interne hooks** (Gateway-hooks): gebeurtenisgestuurde scripts voor opdrachten en levenscyclusgebeurtenissen.
- **Plugin-hooks**: uitbreidingspunten binnen de levenscyclus van agents/tools en de Gateway-pijplijn.

### Interne hooks (Gateway-hooks)

- **`agent:bootstrap`**: wordt uitgevoerd tijdens het opbouwen van bootstrapbestanden, voordat de systeemprompt definitief wordt gemaakt. Gebruik deze hook om bootstrapcontextbestanden toe te voegen of te verwijderen.
- **Opdrachthooks**: `/new`, `/reset`, `/stop` en andere opdrachtgebeurtenissen (zie de documentatie over hooks).

Zie [Hooks](/nl/automation/hooks) voor configuratie en voorbeelden.

### Plugin-hooks

Deze worden uitgevoerd binnen de agentlus of Gateway-pijplijn:

| Hook                                                    | Wordt uitgevoerd                                                                                                                                                                                                                                                                             |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `before_model_resolve`                                  | Voorafgaand aan de sessie (zonder `messages`), om de provider/het model vóór de bepaling deterministisch te overschrijven.                                                                                                                                                                  |
| `before_prompt_build`                                   | Na het laden van de sessie (met `messages`), om vóór de indiening `prependContext`, `systemPrompt`, `prependSystemContext` of `appendSystemContext` te injecteren. Gebruik `prependContext` voor dynamische tekst per beurt en de systeemcontextvelden voor stabiele instructies die in de systeemprompt thuishoren. |
| `before_agent_reply`                                    | Na inline-acties, vóór de LLM-aanroep. Hiermee kan een Plugin de beurt overnemen en een synthetisch antwoord retourneren of de beurt volledig stilhouden.                                                                                                                                     |
| `agent_end`                                             | Na voltooiing, met de definitieve berichtenlijst en uitvoeringsmetadata.                                                                                                                                                                                                                      |
| `before_compaction` / `after_compaction`                | Observeert of annoteert Compaction-cycli.                                                                                                                                                                                                                                                     |
| `before_tool_call` / `after_tool_call`                  | Onderschept toolparameters/-resultaten.                                                                                                                                                                                                                                                       |
| `before_install`                                        | Nadat het installatiebeleid van de operator is uitgevoerd, op klaargezet Skill-/Plugin-installatiemateriaal, wanneer Plugin-hooks in het huidige proces zijn geladen.                                                                                                                         |
| `tool_result_persist`                                   | Transformeert toolresultaten synchroon voordat ze naar een sessietranscript in beheer van OpenClaw worden geschreven.                                                                                                                                                                         |
| `message_received` / `message_sending` / `message_sent` | Hooks voor inkomende en uitgaande berichten.                                                                                                                                                                                                                                                  |
| `session_start` / `session_end`                         | Grenzen van de sessielevenscyclus.                                                                                                                                                                                                                                                            |
| `gateway_start` / `gateway_stop`                        | Levenscyclusgebeurtenissen van de Gateway.                                                                                                                                                                                                                                                    |

Beslisregels voor hooks voor uitgaande berichten/toolbeveiliging:

- `before_tool_call`: `{ block: true }` is terminaal en stopt handlers met een lagere prioriteit. `{ block: false }` doet niets en heft een eerdere blokkering niet op.
- `before_install`: dezelfde semantiek voor terminale waarden en waarden die niets doen als hierboven. Gebruik `security.installPolicy`, niet `before_install`, voor installatiebeslissingen van de operator over toestaan/blokkeren die de installatie- en updatepaden van de CLI moeten omvatten.
- `message_sending`: `{ cancel: true }` is terminaal en stopt handlers met een lagere prioriteit. `{ cancel: false }` doet niets en heft een eerdere annulering niet op.

Zie [Plugin-hooks](/nl/plugins/hooks) voor de hook-API en registratiedetails.

Harnassen kunnen deze hooks aanpassen. Het harnas van de Codex-appserver behoudt de Plugin-hooks van OpenClaw als compatibiliteitscontract voor gedocumenteerde gespiegelde oppervlakken; systeemeigen Codex-hooks zijn een afzonderlijk Codex-mechanisme op lager niveau.

## Streaming

- Assistentdelta's worden vanuit de agentruntime gestreamd als `assistant`-gebeurtenissen.
- Blokstreaming kan gedeeltelijke antwoorden verzenden bij `text_end` of `message_end`.
- Het streamen van redeneringen kan een afzonderlijke stream vormen of antwoorden blokkeren.
- Zie [Streaming](/nl/concepts/streaming) voor het opdelen in fragmenten en het gedrag van blokantwoorden.

## Tooluitvoering

- Gebeurtenissen voor het starten/bijwerken/beëindigen van tools worden op de `tool`-stream verzonden.
- Toolresultaten worden vóór loggen/verzenden opgeschoond wat betreft grootte en afbeeldingspayloads.
- Verzendingen met berichtentools worden bijgehouden om dubbele bevestigingen van de assistent te onderdrukken.

## Vormgeving van antwoorden

Definitieve payloads worden samengesteld uit assistenttekst (plus optionele redenering), inline-toolsamenvattingen (wanneer uitgebreid en toegestaan) en fouttekst van de assistent wanneer het model een fout oplevert.

- Het exacte stille token `NO_REPLY` wordt uit uitgaande payloads gefilterd.
- Duplicaten van berichtentools worden uit de definitieve payloadlijst verwijderd.
- Als er geen renderbare payloads overblijven en een tool een fout heeft opgeleverd, wordt een terugvalantwoord met de toolfout verzonden, tenzij een berichtentool al een voor de gebruiker zichtbaar antwoord heeft verzonden.

## Compaction en nieuwe pogingen

Automatische Compaction verzendt `compaction`-streamgebeurtenissen en kan een nieuwe poging activeren. Bij een nieuwe poging worden buffers in het geheugen en toolsamenvattingen opnieuw ingesteld om dubbele uitvoer te voorkomen. Zie [Compaction](/nl/concepts/compaction).

## Gebeurtenisstreams

- `lifecycle`: verzonden door `subscribeEmbeddedAgentSession` (en als terugval door `agentCommand`).
- `assistant`: gestreamde delta's vanuit de agentruntime.
- `tool`: gestreamde toolgebeurtenissen vanuit de agentruntime.

De Gateway projecteert levenscyclusgebeurtenissen en start-/terminale toolgebeurtenissen naar het begrensde,
uitsluitend uit metadata bestaande [auditlogboek](/nl/cli/audit). Deze projectie registreert herkomst en
resultaatcodes zonder prompts, berichten, toolargumenten, toolresultaten
of onbewerkte fouten uit het transcript-/runtimepad te kopiëren.

## Afhandeling van chatkanalen

Assistentdelta's worden gebufferd in `delta`-chatberichten. Er wordt een `final`-chatbericht verzonden bij **einde/fout van levenscyclus**.

## Time-outs

| Time-out                                          | Standaardwaarde                        | Opmerkingen                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent.wait`                                     | 30s                                    | Alleen wachten; de parameter `timeoutMs` overschrijft dit. Stopt de onderliggende uitvoering niet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Agent-runtime (`agents.defaults.timeoutSeconds`) | 172800s (48h)                          | Afgedwongen door de afbreektimer van `runEmbeddedAgent`. Stel `0` in voor een onbeperkt uitvoeringsbudget; bewakingsmechanismen voor de activiteit van de modelstream blijven van toepassing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Bewakingsmechanisme voor geen uitvoer van de CLI-backend | berekend per nieuwe/hervatte CLI-uitvoering | Staat los van de agent-runtime en valt onder de verantwoordelijkheid van de geregistreerde backendplugin. Een interne CLI-achtergrondtaak deelt het bovenliggende subproces en blijft niet actief na een algemene agent-time-out.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Geïsoleerde agentbeurt van Cron                   | beheerd door Cron                       | De planner start een eigen timer wanneer de uitvoering begint, breekt de uitvoering af bij de geconfigureerde deadline en voert vervolgens begrensde opschoning uit voordat de time-out wordt vastgelegd, zodat een verouderde kindsessie de uitvoeringsbaan niet geblokkeerd kan houden.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Time-out voor inactiviteit van het model          | Cloud 120s; zelfgehost 300s             | OpenClaw breekt een modelverzoek af wanneer vóór het einde van het inactiviteitsvenster geen antwoordfragmenten binnenkomen. `models.providers.<id>.timeoutSeconds` verlengt dit bewakingsmechanisme voor inactiviteit voor trage lokale/zelfgehoste providers, maar blijft begrensd door een eventuele lagere eindige `agents.defaults.timeoutSeconds` of uitvoeringsspecifieke time-out, omdat die de volledige agentuitvoering bepalen. Bij onbeperkte uitvoeringsbudgetten blijft het bewakingsmechanisme voor inactiviteit van de providerklasse actief. Door Cron geactiveerde cloudmodeluitvoeringen zonder expliciete model-/agent-time-out gebruiken dezelfde standaardwaarde; met een expliciete time-out voor de Cron-uitvoering worden vastgelopen cloudmodelstreams begrensd op 60s, zodat geconfigureerde model-fallbacks nog vóór de buitenste Cron-deadline kunnen worden uitgevoerd. Door Cron geactiveerde uitvoeringen op daadwerkelijk lokale eindpunten (loopback/private baseUrl) behouden de lokale uitschakeling van de inactiviteitstime-out; zelfgehoste providers met baseUrls op het netwerk krijgen het impliciete bewakingsmechanisme van 300s. Met een expliciete time-out voor de Cron-uitvoering worden vastlopers van lokale/zelfgehoste providers begrensd op die time-out. Stel `models.providers.<id>.timeoutSeconds` in voor trage lokale providers. |
| Time-out voor HTTP-verzoeken van providers        | `models.providers.<id>.timeoutSeconds` | Omvat verbinding, headers, body, time-out voor SDK-verzoeken, afbreekverwerking van guarded-fetch en het bewakingsmechanisme voor inactiviteit van de modelstream voor die provider. Gebruik dit voor trage lokale/zelfgehoste providers (bijvoorbeeld Ollama) voordat je de time-out van de volledige agent-runtime verhoogt; houd de time-out van de agent/runtime minstens even hoog wanneer het modelverzoek langer moet kunnen worden uitgevoerd.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

### Diagnostiek voor vastgelopen sessies

Als diagnostiek is ingeschakeld, classificeert een ingebouwde drempel van twee minuten langdurige `processing`-sessies waarin geen antwoord-, tool-, status-, blokkerings- of ACP-voortgang is waargenomen:

- Actieve ingesloten uitvoeringen, modelaanroepen en toolaanroepen worden gerapporteerd als `session.long_running`. Stille modelaanroepen met een eigenaar blijven `session.long_running` tot de afbreekdrempel, zodat trage of niet-streamende providers niet te vroeg als vastgelopen worden gemarkeerd.
- Actief werk zonder recente voortgang wordt gerapporteerd als `session.stalled`. Modelaanroepen met een eigenaar schakelen op of na de afbreekdrempel over naar `session.stalled`; verouderde model-/toolactiviteit zonder eigenaar wordt niet verborgen als langdurig actief.
- `session.stuck` is voorbehouden aan herstelbare verouderde sessieadministratie, waaronder inactieve sessies in de wachtrij met verouderde model-/toolactiviteit zonder eigenaar.

De afbreekdrempel bedraagt minstens 5 minuten en 3x de waarschuwingsdrempel. Verouderde sessieadministratie geeft de betrokken sessiebaan onmiddellijk vrij nadat de herstelcontroles zijn geslaagd; vastgelopen ingesloten uitvoeringen worden pas na de afbreekdrempel afgebroken en leeggemaakt, zodat werk in de wachtrij wordt hervat zonder slechts trage uitvoeringen af te kappen. Herstel produceert gestructureerde aangevraagde/voltooide resultaten; de diagnostische status wordt alleen als inactief gemarkeerd als dezelfde verwerkingsgeneratie nog steeds actueel is, en herhaalde `session.stuck`-diagnostiek wordt steeds minder vaak uitgevoerd zolang de sessie ongewijzigd blijft.

## Waar processen voortijdig kunnen eindigen

- Agent-time-out (afbreken)
- AbortSignal (annuleren)
- Verbinding met Gateway verbroken of RPC-time-out
- Time-out van `agent.wait` (alleen wachten, stopt de agent niet)

## Gerelateerd

- [Tools](/nl/tools) - beschikbare agenttools
- [Hooks](/nl/automation/hooks) - gebeurtenisgestuurde scripts die door levenscyclusgebeurtenissen van de agent worden geactiveerd
- [Compaction](/nl/concepts/compaction) - hoe lange gesprekken worden samengevat
- [Uitvoeringsgoedkeuringen](/nl/tools/exec-approvals) - goedkeuringscontroles voor shell-opdrachten
- [Denken](/nl/tools/thinking) - configuratie van het denk-/redeneerniveau
