---
read_when:
    - Je moet sessie-id's, transcriptgebeurtenissen of velden van sessierijen debuggen
    - Je wijzigt het gedrag van automatische compaction of voegt opschoning vóór compaction toe
    - Je wilt geheugenflushes of stille systeembeurten implementeren
summary: 'Diepgaand: sessieopslag en transcripties, levenscyclus en interne werking van (automatische) Compaction'
title: Diepgaande uitleg over sessiebeheer
x-i18n:
    generated_at: "2026-07-27T06:32:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ae02d49245768831abd17e1c2e5adacfa1a36673cef2a8a7a06a5300392b104
    source_path: reference/session-management-compaction.md
    workflow: 16
---

Eén **Gateway-proces** beheert de sessiestatus van begin tot eind. UI's (macOS-app, web-Control UI, TUI) vragen de Gateway om sessielijsten en aantallen tokens. In de externe modus bevinden sessiebestanden zich op de externe host, waardoor controle van de bestanden op je lokale Mac niet weergeeft wat de Gateway gebruikt.

Lees eerst de overzichtsdocumentatie: [Sessiebeheer](/nl/concepts/session), [Compaction](/nl/concepts/compaction), [Geheugenoverzicht](/nl/concepts/memory), [Zoeken in geheugen](/nl/concepts/memory-search), [Sessies opschonen](/nl/concepts/session-pruning), [Transcripthygiëne](/nl/reference/transcript-hygiene), en de volledige configuratiereferentie bij [Agentconfiguratie](/nl/gateway/config-agents).

## Twee persistentielagen

1. **Sessierijen (SQLite per agent)** - sleutel-waardetoewijzing `sessionKey -> SessionEntry`. Veranderlijke runtimestatus die door de Gateway wordt beheerd. Houdt metagegevens bij: huidige sessie-id, laatste activiteit, schakelaars en tokentellers.
2. **Transcriptgebeurtenissen (SQLite per agent)** - alleen-toevoegen en boomgestructureerd (vermeldingen hebben `id` + `parentId`). Slaat het gesprek, toolaanroepen en Compaction-samenvattingen op; reconstrueert modelcontext voor toekomstige beurten. Compaction-controlepunten zijn metagegevens over het gecompacteerde vervolgtranscript: een nieuwe Compaction schrijft geen tweede kopie van `.checkpoint.*.jsonl`.

Oudere installaties bevatten mogelijk nog `sessions.json`-bestanden in de map `sessions/`
van de agent. Behandel deze bestanden als invoer voor de migratie van verouderde sessierijen of als expliciete
doelen voor offline onderhoud. Bij het starten van de Gateway en via `openclaw doctor --fix` worden
actieve verouderde rijen en transcriptgeschiedenis automatisch geïmporteerd in de SQLite-opslag
per agent. Voer `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` uit en volg daarna de [Doctor-migratiereeks
](/nl/cli/doctor#session-sqlite-migration) wanneer je expliciet
inspectie- of validatiebewijs nodig hebt. Als een migratie mislukt nadat verouderde transcriptartefacten
zijn gearchiveerd, gebruik je de Doctor-herstelmodus uit die reeks.
Herstel gebruikt migratiemanifesten, herstelt alleen de betrokken gearchiveerde ondersteuningsartefacten,
stelt op verzoek een opgeschoond GitHub-issuemeldingsrapport op en zorgt er niet voor
dat de actieve runtime opnieuw JSONL-bestanden leest.

Geschiedenislezers van de Gateway laden niet het volledige transcript in het geheugen, tenzij het oppervlak willekeurige historische toegang nodig heeft. Geschiedenis op de eerste pagina, ingesloten chatgeschiedenis, herstel na opnieuw starten en token-/gebruikscontroles gebruiken begrensde uitlezingen van het einde uit SQLite. Volledige transcriptscans verlopen via de asynchrone transcriptindex en worden gedeeld tussen gelijktijdige lezers.

## Locaties op schijf

Per agent, op de Gateway-host (bepaald via `src/config/sessions.ts`):

- Runtimeopslag voor sessierijen: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Runtimetranscriptrijen: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- Verouderde/gearchiveerde transcriptartefacten: `~/.openclaw/agents/<agentId>/sessions/`
- Migratie-invoer voor verouderde rijen: `~/.openclaw/agents/<agentId>/sessions/sessions.json`

## Opslagonderhoud en schijflimieten

`session.maintenance` regelt automatisch onderhoud voor SQLite-sessierijen, SQLite-transcriptrijen, archiefartefacten en traject-zijbestanden:

| Sleutel                 | Standaard             | Opmerkingen                                                                                 |
| ----------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| `mode`                  | `"enforce"`           | of `"warn"` (alleen rapporteren, geen wijzigingen)                                         |
| `pruneAfter`            | `"30d"`               | leeftijdsgrens voor verouderde vermeldingen                                                  |
| `maxEntries`            | `500`                 | maximumaantal sessievermeldingen                                                             |
| `resetArchiveRetention` | behouden (geen leeftijdsgrens) | leeftijdsgrens voor `*.reset.*`-/`*.deleted.*`-transcriptarchieven; een duur schakelt verwijdering in |
| `maxDiskBytes`          | `10gb`                | schijfbudget voor sessies per agent; `false` schakelt dit uit                               |
| `highWaterBytes`        | 80% van `maxDiskBytes` | doel na opschoning voor het budget                                                           |

Opnieuw instellen schuift de actieve `sessionKey -> sessionId`-toewijzing door, maar behoudt de vorige SQLite-sessie-, transcript-, traject- en zoekrijen. Die geschiedenis blijft doorzoekbaar onder dezelfde sessiesleutel; normale vermeldings- en sessielijsten tonen alleen de nieuwe actieve toewijzing. Behouden geschiedenis van opnieuw ingestelde sessies wordt begrensd door het schijfbudget, niet door `resetArchiveRetention`, dat alleen archiefartefacten op leeftijd verwijdert. Expliciet verwijderen werkt anders: daarbij wordt een gecomprimeerd transcriptarchief geschreven en geverifieerd (`*.jsonl.deleted.<timestamp>.zst` wanneer zstd beschikbaar is), voordat de rijen van de verwijderde sessie worden verwijderd.

Afdwinging van `maxDiskBytes` gebruikt fysieke bytes: het SQLite-hoofdbestand per agent, het bijbehorende `-wal`-bestand en meegetelde bestanden in de sessiemap van de agent. De JSON-grootte van rijen wordt nooit geschat en logische rijgroottes worden nooit van dit totaal afgetrokken.

Probesessies voor modelruns van de Gateway (sleutels die overeenkomen met `agent:*:explicit:model-run-<uuid>`) krijgen een afzonderlijke, vaste bewaartermijn van `24h`. Dit opschonen is afhankelijk van druk: het wordt alleen uitgevoerd wanneer onderhouds- of capaciteitsdruk voor sessievermeldingen is bereikt, en alleen vóór de algemene stap voor het opschonen of begrenzen van verouderde vermeldingen. Andere expliciete sessies gebruiken deze bewaartermijn niet.

Wanneer het gecombineerde fysieke gebruik hoger is dan `maxDiskBytes`, maakt `mode: "enforce"` eerst databaseopslag vrij waarvoor een controlepunt kan worden gemaakt en verwijdert daarna de oudste behouden archieven van opnieuw ingestelde of verwijderde sessies. Als het gebruik nog steeds hoger is dan `highWaterBytes`, doorloopt het historische SQLite-sessies op basis van `sessions.updated_at`, met de oudste eerst. Historisch betekent dat de sessie-id niet wordt verwezen door een actieve sessievermelding, een routedoel of een toegelaten/lopende uitvoering. Voor elk slachtoffer schrijft de opschoning het gecomprimeerde archief, voert fsync uit en leest het terug, voordat een schrijftransactie de sessierij en de bijbehorende transcript-, traject-, actieve, index- en FTS-projecties verwijdert. Dit omvat sessies die trajectgebeurtenissen bevatten maar geen transcriptgebeurtenissen. De opschoning controleert route-, vermeldings- en toelatingsverwijzingen opnieuw op het moment van verwijderen, meet het fysieke gebruik na elk archief of elke verwijderde sessie opnieuw en stopt bij `highWaterBytes`.

Vastgelegde schrijfbewerkingen en verwijderingen komen eerst in de WAL terecht. De opschoning maakt er een controlepunt van zodat de WAL direct kan krimpen en gebruikt daarna incrementele vacuum om beschikbare vrije eindpagina's uit het hoofdbestand terug te geven; pagina's die nog niet kunnen worden vrijgemaakt, blijven in het hoofdbestand en worden daarom bij de volgende fysieke meting meegeteld. `mode: "warn"` rapporteert de huidige fysieke overschrijding zonder een controlepunt te maken, een archief te schrijven of rijen te verwijderen.

Voer onderhoud op aanvraag uit:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

Onderhoud behoudt duurzame externe gespreksverwijzingen, zoals groepssessies en threadgebonden chatsessies, maar synthetische runtimevermeldingen (Cron, hooks, Heartbeat, ACP, subagenten) kunnen nog steeds worden verwijderd zodra ze de geconfigureerde leeftijd, het aantal of het schijfbudget overschrijden. Geïsoleerde Cron-uitvoeringen gebruiken een afzonderlijke instelling `cron.sessionRetention`, onafhankelijk van de bewaartermijn voor modelrunprobes.

Normale schrijfbewerkingen van de Gateway verlopen via de sessie-accessor, die SQLite-wijzigingen per agent serialiseert via het runtimeschrijfpad. Runtimecode hoort bij voorkeur de accessorhelpers in `src/config/sessions/session-accessor.ts` te gebruiken; de verouderde `sessions.json`-helpers zijn tools voor migratie en offline onderhoud. Wanneer een Gateway bereikbaar is, delegeren niet-droog uitgevoerde `openclaw sessions cleanup` en `openclaw agents delete` opslagwijzigingen aan de Gateway, zodat de opschoning aan dezelfde schrijfqueue wordt toegevoegd; `--store <path>` is het expliciete offline herstelpad voor een geselecteerde verouderde opslag en blijft altijd lokaal (net als `--dry-run`). Opschoning van `maxEntries` wordt in batches uitgevoerd voor opslagen van productieomvang, waardoor een opslag kortstondig het geconfigureerde maximum kan overschrijden voordat de volgende opschoning bij de bovengrens de opslag weer tot onder het maximum terugbrengt. Lezen snoeit of begrenst vermeldingen nooit tijdens het starten van de Gateway; alleen schrijfbewerkingen of `openclaw sessions cleanup --enforce` doen dit. Die laatste past het maximum bovendien onmiddellijk toe en verwijdert oude, niet-verwezen verouderde transcript-, controlepunt- en trajectartefacten, zelfs als er geen schijfbudget is geconfigureerd.

OpenClaw maakt niet langer automatisch rotatieback-ups van `sessions.json.bak.*` tijdens schrijfbewerkingen van de Gateway. Het huidige schema weigert de verouderde sleutel `session.maintenance.rotateBytes` en `openclaw doctor --fix` verwijdert deze uit oudere configuraties.

Transcriptwijzigingen gebruiken de schrijfwachtrij van de sessie voor het SQLite-transcriptdoel:

Schrijfvergrendelingen voor sessies gebruiken vaste productiestandaarden. De bijbehorende
`OPENCLAW_SESSION_WRITE_LOCK_*`-omgevingsvariabelen blijven beschikbaar voor
diagnostiek op procesniveau en noodoverschrijvingen.

### Downgraden na de overstap naar SQLite

Herstel gearchiveerde verouderde transcriptartefacten voordat je een oudere,
bestandsgebaseerde versie van OpenClaw uitvoert:

```bash
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

De migratie laat verouderde `sessions.json`-bestanden op hun plek staan voor ondersteuning en
terugdraaien, maar actieve JSONL-transcriptbestanden die in SQLite zijn geïmporteerd,
worden hernoemd naar `session-sqlite-import-archive/`. Oudere bestandsgebaseerde runtimes volgen
de `sessionFile`-paden in `sessions.json` en moeten deze artefacten daarom vóór
het starten herstellen. Het herstel gebruikt migratiemanifesten, verplaatst alleen vastgelegde gearchiveerde
artefacten waarvan de oorspronkelijke paden ontbreken en laat de SQLite-database staan
voor toekomstig herstel.

Sessies die na de overstap naar SQLite zijn gemaakt, bestaan alleen in SQLite en zijn niet zichtbaar voor een
oudere bestandsgebaseerde runtime. Als je na een downgrade opnieuw upgradet, voer je de
inspectie- en validatiereeks van Doctor opnieuw uit, zodat OpenClaw de herstelde verouderde
artefacten kan verifiëren voordat ze worden geïmporteerd.

## Cron-sessies en uitvoeringslogboeken

Geïsoleerde Cron-uitvoeringen maken hun eigen sessievermeldingen/transcripten met een afzonderlijk bewaarbeleid:

- `cron.sessionRetention` (standaard `"24h"`) verwijdert oude sessies van geïsoleerde Cron-uitvoeringen uit de opslag; `false` schakelt dit uit.
- De uitvoeringsgeschiedenis bewaart de nieuwste 2000 eindstatusrijen per Cron-taak. Verloren rijen behouden hun opschoningsvenster van 24 uur.

Wanneer Cron geforceerd een nieuwe geïsoleerde uitvoeringssessie maakt, schoont het de vorige `cron:<jobId>`-sessievermelding op voordat de nieuwe rij wordt geschreven: veilige voorkeuren (instellingen voor denken/snel/uitgebreid/redeneren, labels en weergavenaam) en expliciet door de gebruiker geselecteerde model-/authenticatieoverschrijvingen worden meegenomen, maar omgevingscontext van gesprekken (kanaal-/groepsroutering, verzend-/wachtrijbeleid, verhoogde bevoegdheden, oorsprong en ACP-runtimebinding) wordt verwijderd, zodat een nieuwe geïsoleerde uitvoering geen verouderde afleverings- of runtimebevoegdheid van een oudere uitvoering kan erven.

## Sessiesleutels (`sessionKey`)

Een `sessionKey` bepaalt in welke gesprekscontainer je zit (routering + isolatie). Canonieke regels: [/concepts/session](/nl/concepts/session).

| Patroon                      | Voorbeeld                                                   |
| ---------------------------- | ----------------------------------------------------------- |
| Hoofd-/directe chat (per agent) | `agent:<agentId>:<mainKey>` (standaard `main`)             |
| Groep                        | `agent:<agentId>:<channel>:group:<id>`                                          |
| Ruimte/kanaal (Discord/Slack) | `agent:<agentId>:<channel>:channel:<id>` of `...:room:<id>`                    |
| Cron                         | `cron:<job.id>`                                          |
| Webhook                      | `hook:<uuid>` (tenzij overschreven)                    |

## Sessie-id's (`sessionId`)

Elke `sessionKey` verwijst naar een huidige `sessionId` (de SQLite-transcriptidentiteit die het gesprek voortzet). De beslislogica bevindt zich in `initSessionState()` in `src/auto-reply/reply/session.ts`.

- **Resetten** (`/new`, `/reset`) maakt een nieuwe `sessionId` voor die `sessionKey`.
- **Geen automatische reset** is de standaardinstelling. De huidige `sessionId` gaat door terwijl Compaction de actieve modelcontext begrensd houdt.
- **Dagelijkse reset** (`session.reset.mode: "daily"`) maakt een nieuwe `sessionId` bij het eerstvolgende bericht na de geconfigureerde grens op basis van het lokale uur (`session.reset.atHour`, standaard `4`).
- **Verloop na inactiviteit** (`session.reset.mode: "idle"` met `session.reset.idleMinutes`, of de verouderde `session.idleMinutes`) maakt een nieuwe `sessionId` wanneer een bericht na het inactiviteitsvenster binnenkomt. Als zowel dagelijks als na inactiviteit is geconfigureerd, geldt wat het eerst verloopt.
- **Hervatten na opnieuw verbinden van de bedieningsinterface** behoudt de momenteel zichtbare sessie voor één verzending na opnieuw verbinden wanneer de Gateway de overeenkomende `sessionId` van een UI-client van een operator ontvangt. Dit is een eenmalig signaal; gewone verouderde verzendingen maken nog steeds een nieuwe `sessionId`.
- **Systeemgebeurtenissen** (Heartbeat, Cron-activeringen, uitvoeringsmeldingen, Gateway-administratie) kunnen de sessierij wijzigen, maar verlengen nooit de geldigheid voor dagelijkse resets of resets na inactiviteit. Bij het overschakelen na een reset worden meldingen van systeemgebeurtenissen in de wachtrij voor de vorige sessie verwijderd voordat de nieuwe prompt wordt opgebouwd.
- **Beleid voor forks van bovenliggende sessies** gebruikt de actieve tak van OpenClaw bij het maken van een thread- of subagentfork. Als die tak te groot is (boven een vaste interne limiet, momenteel 100K tokens), start OpenClaw het onderliggende proces met een geïsoleerde context in plaats van te mislukken of onbruikbare geschiedenis over te nemen. De groottebepaling verloopt automatisch en is niet configureerbaar; de verouderde configuratie `session.parentForkMaxTokens` wordt verwijderd door `openclaw doctor --fix`.
- **Operatorforks**: `sessions.create { parentSessionKey, fork: true }` maakt een nieuwe sessie waarvan het transcript aftakt van de huidige toestand van de bovenliggende sessie (hetzelfde forkmechanisme als bij het starten van subagents, inclusief de bovenstaande groottelimiet). De fork wordt geweigerd zolang de bovenliggende sessie een actieve uitvoering heeft, neemt de modelselectie van de bovenliggende sessie over tenzij er expliciet een wordt doorgegeven, en markeert de onderliggende `forkedFromParent` met nieuwe tokentellers.

## Schema van de sessieopslag

De runtimeopslag bewaart `SessionEntry`-waarden in SQLite per agent. Het waardetype is `SessionEntry` in `src/config/sessions.ts`. Belangrijke velden (niet uitputtend):

- `sessionId`: huidige transcript-id waarmee SQLite-transcriptrijen worden geadresseerd
- `sessionStartedAt`: begintijdstempel voor de huidige `sessionId`; de geldigheid voor dagelijkse resets gebruikt deze. Verouderde rijen kunnen deze afleiden uit de JSONL-sessieheader.
- `lastInteractionAt`: tijdstempel van de laatste echte interactie met een gebruiker of kanaal; de geldigheid voor resets na inactiviteit gebruikt deze, zodat Heartbeat-, Cron- en uitvoeringsgebeurtenissen sessies niet actief houden. Verouderde rijen zonder dit veld vallen terug op de herstelde begintijd van de sessie.
- `updatedAt`: tijdstempel van de laatste wijziging van de opslagrij, gebruikt voor opsommen, opschonen en administratie — niet de bepalende bron voor de geldigheid van dagelijkse resets of resets na inactiviteit.
- `archivedAt`: optionele archieftijdstempel. Gearchiveerde sessies blijven met hun transcript intact in de opslag en worden uitgesloten van normale lijsten met actieve sessies.
- `pinnedAt`: optionele vastzettijdstempel. Actieve vastgezette sessies worden vóór niet-vastgezette sessies gesorteerd; bij het archiveren van een sessie wordt de vastzetting opgeheven.
- Interoperabiliteit met Codex-threads: beide velden volgen de vorm voor threadbeheer van Codex — de booleaanse waarden `archived`/`pinned` op de verbinding worden altijd afgeleid van de tijdstempel en aan serverzijde vastgelegd, overeenkomstig de semantiek van Codex `threads.archived_at` en serialisatie in camelCase. OpenClaw-tijdstempels zijn epoch-milliseconden, terwijl Codex epoch-seconden gebruikt, dus bridges converteren bij de `codex`-Plugininterface. Codex heeft nog geen API voor vastzetten (alleen `thread/archive`/`thread/unarchive`); de vastgezette toestand blijft aan OpenClaw-zijde totdat er een bestaat. Op dat moment kan de overeenkomende vorm de vastgezette toestand van gekoppelde sessies mechanisch heen en terug overbrengen.
- Codex-supervisie vermeldt alleen niet-gearchiveerde native threads. Een Gateway-lokale thread met `idle` of `notLoaded` waarvan de activiteit onbekend is, kan alleen via native `thread/archive` worden gearchiveerd nadat de operator expliciet heeft bevestigd dat geen ander Codex-proces de eigenaar is; de Plugin leest eerst opnieuw de proceslokale status, waarna de thread uit de catalogus verdwijnt. Die uitlezing kan niet bewijzen dat een ander App Server-proces de thread niet gebruikt. OpenClaw weigert actieve rijen en foutrijen te archiveren, en archivering via een gekoppelde Node is niet beschikbaar totdat de Node-bridge de volledige gestreamde levenscyclus van de thread kan beheren. Als de archivering in een native Codex-client ongedaan wordt gemaakt, kan de thread weer worden weergegeven.
- `lastReadAt` / `markedUnreadAt`: tijdstempels van de leesstatus die aan serverzijde worden vastgelegd door `sessions.patch { unread }` — `unread: false` registreert dat iets is gelezen (stelt `lastReadAt` in en wist `markedUnreadAt`); `unread: true` markeert de sessie als ongelezen tot de volgende keer dat deze wordt gelezen. Sessierijen tonen een afgeleide booleaanse waarde `unread`: expliciet gemarkeerd als ongelezen, of gelezen vóór de recentste activiteit. Sessies die nooit als gelezen zijn gemarkeerd, blijven `unread: false`, zodat bestaande installaties na een upgrade niet plotseling oplichten.
- `lastActivityAt`: tijdstempel van de laatst voltooide agentuitvoering die als noemenswaardige ongelezen activiteit telt (uitvoeringen door gebruikers, kanalen en Cron). Heartbeat- en interne-gebeurtenisbeurten en metadatapatches werken deze niet bij; `updatedAt` is geen activiteitssignaal.
- `sessionFile`: verouderde markering die behouden blijft voor compatibiliteit met migratie en archivering; de actieve runtime gebruikt SQLite-identiteit
- `chatType`: `direct | group | room`
- `provider`, `subject`, `room`, `space`, `displayName`: metadata voor groeps- en kanaallabels
- Schakelaars: `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`, `sendPolicy` (overschrijving per sessie)
- Modelselectie: `providerOverride`, `modelOverride`, `authProfileOverride`
- Tokentellers (naar beste vermogen/afhankelijk van de provider): `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: het aantal keren dat automatische Compaction voor deze sessiesleutel is voltooid
- `memoryFlushAt` / `memoryFlushCompactionCount`: tijdstempel en Compaction-telling van de laatste geheugenflush vóór Compaction

De Gateway is de autoriteit: deze kan items herschrijven of opnieuw hydrateren terwijl sessies
worden uitgevoerd. Migreer verouderde installaties met bestandsopslag met
`openclaw doctor --session-sqlite import --session-sqlite-all-agents` in plaats van
`sessions.json` te bewerken en te verwachten dat de runtime dat bestand blijft lezen.

## Structuur van transcriptgebeurtenissen

Transcripten worden beheerd door de sessietoegangslaag van OpenClaw en via op identiteit gebaseerde helpers beschikbaar gesteld aan runtimecode. De gebeurtenissenstroom is alleen-toevoegen:

- Eerste item: sessieheader — `type: "session"`, `id`, `cwd`, `timestamp`, optioneel `parentSession`.
- Daarna: items met `id` + `parentId` (boomstructuur).

Opmerkelijke itemtypen:

- `message`: berichten van gebruiker/assistent/toolResult
- `custom_message`: door een extensie geïnjecteerd bericht dat _wel_ in de modelcontext terechtkomt (weergegeven in de TUI wanneer `display: true`, volledig verborgen wanneer `display: false`)
- `custom`: extensietoestand die _niet_ in de modelcontext terechtkomt (om de extensietoestand tussen herlaadacties te bewaren)
- `compaction`: persistente Compaction-samenvatting met `firstKeptEntryId` en `tokensBefore`
- `branch_summary`: persistente samenvatting bij het navigeren door een tak van de boom

OpenClaw corrigeert transcripten bewust niet; de Gateway gebruikt `SessionManager` om ze te lezen en te schrijven.

## Contextvensters versus bijgehouden tokens

Twee verschillende concepten:

1. **Modelcontextvenster**: harde limiet per model (tokens die zichtbaar zijn voor het model). Komt uit de modelcatalogus en kan via de configuratie worden overschreven.
2. **Tellers in de sessieopslag**: doorlopende statistieken die naar de sessierij worden geschreven (gebruikt voor `/status` en dashboards). `contextTokens` is een tijdens runtime geschatte/gerapporteerde waarde — beschouw deze niet als een strikte garantie.

Meer over limieten: [/reference/token-use](/nl/reference/token-use).

## Compaction: wat het is

Compaction vat oudere gesprekken samen in een persistent `compaction`-item in het transcript en houdt recente berichten intact. Na Compaction zien toekomstige beurten de Compaction-samenvatting plus de berichten na `firstKeptEntryId`. Compaction is **persistent**, in tegenstelling tot het opschonen van sessies — zie [/concepts/session-pruning](/nl/concepts/session-pruning).

Ingebedde OpenClaw-Compaction neemt standaard het denkniveau van de sessie over. Stel `agents.defaults.compaction.thinkingLevel` in om een afzonderlijk niveau voor samenvattingsaanroepen te gebruiken; de runtime begrenst dit tot elk concreet Compaction-model of terugvalmodel. Native Compaction van de Codex App Server beheert zijn eigen compact-aanvraag en kan geen denkoverschrijving per Compaction accepteren, dus OpenClaw waarschuwt en laat die instelling aan Codex over.

Het opnieuw invoegen van AGENTS.md-secties na Compaction blijft optioneel via `agents.defaults.compaction.postCompactionSections`. Plugins kunnen via `before_prompt_build` andere promptcontext toevoegen.

### Chunkgrenzen en koppeling van tools

Bij het splitsen van een lang transcript in Compaction-chunks houdt OpenClaw toolaanroepen van de assistent gekoppeld aan de bijbehorende `toolResult`-items:

- Als de splitsing op basis van het tokenaandeel tussen een toolaanroep en het resultaat ervan zou vallen, verschuift OpenClaw de grens naar het bericht met de toolaanroep van de assistent in plaats van het paar te scheiden.
- Als een afsluitend blok met toolresultaten de chunk anders boven de doelgrootte zou brengen, behoudt OpenClaw dat wachtende toolblok en laat het niet-samengevatte uiteinde intact.
- Afgebroken toolaanroepblokken en toolaanroepblokken met fouten houden een wachtende splitsing niet open.

## Wanneer automatische Compaction plaatsvindt

Twee triggers in de ingebedde OpenClaw-agent:

1. **Herstel bij overschrijding**: het model retourneert een fout wegens overschrijding van de context (`request_too_large`, `context length exceeded`, `input exceeds the maximum number of tokens`, `input token count exceeds the maximum number of input tokens`, `input is too long for the model`, `ollama error: context length exceeded` en andere providerspecifieke varianten) — voer Compaction uit en probeer het daarna opnieuw. Wanneer de provider het aantal tokens van de poging rapporteert, geeft OpenClaw dat waargenomen aantal door aan Compaction voor herstel bij overschrijding; als de provider de overschrijding bevestigt maar geen parseerbaar aantal beschikbaar stelt, geeft OpenClaw een synthetisch aantal dat minimaal boven het budget ligt door aan Compaction-engines en diagnostiek. Als herstel bij overschrijding nog steeds mislukt, toont OpenClaw expliciete instructies en behoudt het de huidige sessiekoppeling in plaats van stilzwijgend over te schakelen naar een nieuwe sessie-id — probeer het bericht opnieuw, voer `/compact` uit of voer `/new` uit.
2. **Drempelonderhoud**: na een geslaagde beurt, wanneer de huidige context groter is dan het modelvenster minus de ingebouwde speelruimte van OpenClaw voor prompts en de volgende modeluitvoer.

Buiten deze twee triggers worden nog twee extra beveiligingen uitgevoerd:

- **Lokale Compaction vooraf**: stel `agents.defaults.compaction.maxActiveTranscriptBytes` in (bytes of een tekenreeks zoals `"20mb"`) om lokale Compaction te activeren voordat de volgende run wordt geopend zodra het actieve transcript die grootte bereikt. Dit is een groottelimiet voor de kosten van lokaal heropenen, niet voor onbewerkte archivering: normale semantische Compaction blijft actief en vereist `truncateAfterCompaction`, zodat de gecompacteerde samenvatting een nieuw opvolgend transcript wordt.
- **Tussentijdse voorafcontrole**: stel `agents.defaults.compaction.midTurnPrecheck.enabled: true` in (standaard `false`) om een beveiliging voor de tool-lus toe te voegen. Nadat een toolresultaat is toegevoegd en vóór de volgende modelaanroep, schat OpenClaw de promptbelasting met dezelfde voorafgaande budgetlogica die aan het begin van een beurt wordt gebruikt. Als de context niet meer past, voert de beveiliging geen inline Compaction uit: er wordt een gestructureerd signaal voor tussentijdse voorafcontrole gegenereerd, de huidige promptinzending wordt gestopt en de buitenste run-lus gebruikt het bestaande herstelpad (te grote toolresultaten inkorten wanneer dat volstaat, of de geconfigureerde Compaction-modus activeren en het opnieuw proberen). Werkt met zowel de Compaction-modus `default` als `safeguard`, inclusief door een provider ondersteunde beveiligings-Compaction. Onafhankelijk van `maxActiveTranscriptBytes`: de beveiliging voor bytegrootte wordt uitgevoerd voordat een beurt wordt geopend; de tussentijdse voorafcontrole wordt later uitgevoerd, nadat nieuwe toolresultaten zijn toegevoegd.

## Compaction-instellingen

```json5
{
  agents: {
    defaults: {
      compaction: {
        enabled: true,
        keepRecentTokens: 20000,
      },
    },
  },
}
```

OpenClaw dwingt een ingebouwde reserve af voor ingebedde runs en begrenst deze op basis van het contextvenster van het actieve model, zodat deze niet het volledige promptbudget kan verbruiken. Dit voorkomt dat lokale modellen met een kleine context vanaf het eerste token Compaction uitvoeren, terwijl er genoeg ruimte overblijft voor onderhoud over meerdere beurten, zoals het wegschrijven van het geheugen.

Handmatige `/compact` respecteert een expliciete `agents.defaults.compaction.keepRecentTokens` en behoudt het afkappunt van de runtime voor het recente uiteinde. Zonder een expliciet behoudbudget vormt handmatige Compaction een hard controlepunt en begint de opnieuw opgebouwde context bij de nieuwe samenvatting.

Wanneer `truncateAfterCompaction` is ingeschakeld, roteert OpenClaw het actieve transcript na Compaction naar een gecompacteerde opvolger. Acties voor controlepunten bij vertakken/herstellen gebruiken die gecompacteerde opvolger; verouderde controlepuntbestanden van vóór de Compaction blijven leesbaar zolang ernaar wordt verwezen.

## Inplugbare Compaction-providers

Plugins registreren een Compaction-provider via `registerCompactionProvider()` in de Plugin-API. Wanneer `agents.defaults.compaction.provider` is ingesteld op de id van een geregistreerde provider, delegeert de beveiligingsextensie het samenvatten aan die provider in plaats van aan de ingebouwde `summarizeInStages`-pijplijn.

- `provider`: id van een geregistreerde Compaction-provider-Plugin. Laat dit oningesteld voor standaard LLM-samenvatting. Het instellen van een `provider` dwingt `mode: "safeguard"` af.
- Providers ontvangen dezelfde Compaction-instructies en hetzelfde beleid voor het behouden van identificatoren als het ingebouwde pad, en de beveiliging behoudt na de provideruitvoer nog steeds de context van recente beurten en het achtervoegsel van gesplitste beurten.
- De ingebouwde beveiligingssamenvatting destilleert eerdere samenvattingen opnieuw samen met nieuwe berichten, in plaats van de volledige vorige samenvatting letterlijk te behouden.
- De beveiligingsmodus schakelt standaard kwaliteitscontroles voor samenvattingen in; stel `qualityGuard.enabled: false` in om het opnieuw proberen bij onjuist gevormde uitvoer over te slaan.
- Als de provider mislukt of een leeg resultaat retourneert, valt OpenClaw automatisch terug op ingebouwde LLM-samenvatting. Door de aanroeper expliciet geactiveerde afbreek-/timeoutsignalen worden opnieuw gegenereerd en niet onderdrukt, zodat annulering altijd wordt gerespecteerd.

Bron: `src/plugins/compaction-provider.ts`, `src/agents/agent-hooks/compaction-safeguard.ts`.

## Voor de gebruiker zichtbare oppervlakken

- `/status` in elke chatsessie
- `openclaw status` (CLI)
- `openclaw sessions` / `openclaw sessions --json`
- Gateway-logboeken (`pnpm gateway:watch` of `openclaw logs --follow`): `embedded run auto-compaction start` + `complete`
- Uitgebreide modus: `🧹 Auto-compaction complete` plus het aantal Compaction-bewerkingen

## Stil onderhoud (`NO_REPLY`)

OpenClaw ondersteunt 'stille' beurten voor achtergrondtaken waarbij de gebruiker geen tussentijdse uitvoer mag zien.

- De assistent begint de uitvoer met het exacte stille token `NO_REPLY` / `no_reply` om aan te geven: 'lever geen antwoord aan de gebruiker.' OpenClaw verwijdert/onderdrukt dit in de afleveringslaag.
- Onderdrukking van het exacte stille token is niet hoofdlettergevoelig: `NO_REPLY` en `no_reply` tellen beide wanneer de volledige inhoud uitsluitend uit het stille token bestaat.
- Vanaf `2026.1.10` onderdrukt OpenClaw ook concept-/typestreaming wanneer een gedeeltelijk fragment met `NO_REPLY` begint, zodat stille bewerkingen geen gedeeltelijke uitvoer tijdens de beurt lekken.
- Dit is uitsluitend bedoeld voor echte achtergrondbeurten/beurten zonder aflevering; het is geen snelkoppeling voor gewone uitvoerbare gebruikersverzoeken.

## Geheugen wegschrijven vóór Compaction

Voordat automatische Compaction plaatsvindt, kan OpenClaw een stille agentische beurt uitvoeren die duurzame status naar schijf schrijft (bijvoorbeeld `memory/YYYY-MM-DD.md` in de agentwerkruimte), zodat Compaction geen kritieke context kan wissen. Het bewaakt het contextgebruik van de sessie en zodra dit een zachte drempel onder de Compaction-drempel overschrijdt, verzendt het een stille instructie 'geheugen nu schrijven' met het exacte stille token `NO_REPLY` / `no_reply`, zodat de gebruiker niets ziet.

Configuratie (`agents.defaults.compaction.memoryFlush`), volledige referentie op [/gateway/config-agents](/nl/gateway/config-agents#agentsdefaultscompaction):

| Sleutel                     | Standaard        | Opmerkingen                                                                                                                            |
| --------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                   | `true`           |                                                                                                                                        |
| `model`                     | niet ingesteld   | exacte provider-/modeloverschrijving uitsluitend voor de wegschrijfbeurt, bijvoorbeeld `ollama/qwen3:8b`                              |
| `softThresholdTokens`       | `4000`           | afstand onder de Compaction-drempel die het wegschrijven activeert                                                                     |
| `forceFlushTranscriptBytes` | niet ingesteld (uitgeschakeld) | dwing wegschrijven af zodra het transcriptbestand deze bytegrootte bereikt (of een tekenreeks zoals `"2mb"`), zelfs als tokentellers verouderd zijn; `0` schakelt dit uit |

Opmerkingen:

- De ingebouwde prompt en systeemprompt bevatten een `NO_REPLY`-hint om aflevering te onderdrukken.
- Wanneer `model` is ingesteld, gebruikt de wegschrijfbeurt dat model zonder de terugvalketen van de actieve sessie over te nemen, zodat uitsluitend lokaal onderhoud bij een fout niet stilzwijgend terugvalt op een betaald gespreksmodel.
- Het wegschrijven wordt eenmaal per Compaction-cyclus uitgevoerd (bijgehouden in de sessierij).
- Het wegschrijven wordt uitsluitend uitgevoerd voor ingebedde OpenClaw-sessies; CLI-backends en Heartbeat-beurten slaan dit over.
- Het wegschrijven wordt overgeslagen wanneer de sessiewerkruimte alleen-lezen is (`workspaceAccess: "ro"` of `"none"`).
- Zie [Geheugen](/nl/concepts/memory) voor de bestandsindeling van de werkruimte en schrijfpatronen.

OpenClaw stelt een `session_before_compact`-hook beschikbaar in de extensie-API, maar de bovenstaande wegschrijflogica bevindt zich aan de Gateway-zijde (`src/auto-reply/reply/memory-flush.ts`, `src/auto-reply/reply/agent-runner-memory.ts`) en niet in die hook.

## Controlelijst voor probleemoplossing

- **Verkeerde sessiesleutel?** Begin bij [/concepts/session](/nl/concepts/session) en bevestig de `sessionKey` in `/status`.
- **Verschil tussen opslag en transcript?** Bevestig de Gateway-host en het opslagpad uit `openclaw status`.
- **Overmatige Compaction?** Controleer het contextvenster van het model (een te klein venster dwingt frequente Compaction af) en de opgeblazen omvang van toolresultaten (stem het opschonen van sessies af).
- **Lijkt elke prompt over te lopen bij een klein lokaal model?** Bevestig dat de provider het juiste modelcontextvenster rapporteert. OpenClaw kan de effectieve reserve alleen begrenzen wanneer dat venster bekend is.
- **Lekken stille beurten?** Bevestig dat het antwoord begint met het exacte stille token `NO_REPLY` (niet hoofdlettergevoelig) en dat je een build gebruikt die de oplossing voor streamingonderdrukking bevat (`2026.1.10`+).

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session)
- [Sessies opschonen](/nl/concepts/session-pruning)
- [Context-engine](/nl/concepts/context-engine)
- [Referentie voor agentconfiguratie](/nl/gateway/config-agents)
