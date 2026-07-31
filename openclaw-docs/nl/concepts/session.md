---
read_when:
    - Je wilt sessieroutering en -isolatie begrijpen
    - Je wilt het bereik van privéberichten configureren voor configuraties met meerdere gebruikers
    - Je debugt dagelijkse resets of resets van inactieve sessies
summary: Hoe OpenClaw gesprekssessies beheert
title: Sessiebeheer
x-i18n:
    generated_at: "2026-07-27T05:49:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw routeert elk inkomend bericht naar een **sessie** op basis van de
herkomst: privéberichten, groepschats, cron-taken enzovoort. Alle sessiestatus
wordt beheerd door de **Gateway**; UI-clients vragen sessiegegevens op bij de
Gateway.

Zie [De hoofdsessie](/concepts/main-session) voor de standaardinstelling voor
persoonlijke agents: één doorlopend gesprek dat door al je privéberichtkanalen
wordt gedeeld en waarin groepsactiviteit en achtergrondwerk samenkomen.

## Hoe berichten worden gerouteerd

| Bron             | Gedrag                           |
| ---------------- | -------------------------------- |
| Privéberichten   | Standaard gedeelde sessie        |
| Groepschats      | Afzonderlijk per groep            |
| Ruimten/kanalen  | Afzonderlijk per ruimte           |
| Cron-taken       | Nieuwe sessie per uitvoering      |
| Webhooks         | Afzonderlijk per hook             |

## Isolatie van privéberichten

Standaard delen alle privéberichten één sessie om de continuïteit te behouden.
Dat is geschikt voor configuraties met één gebruiker.

<Warning>
Als meerdere personen je agent berichten kunnen sturen, schakel dan isolatie
van privéberichten in. Zonder isolatie delen alle gebruikers dezelfde
gesprekscontext, waardoor de privéberichten van Alice zichtbaar zouden zijn
voor Bob.
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // isoleren op kanaal + afzender
  },
}
```

Opties voor `session.dmScope`:

| Waarde                     | Gedrag                                                         |
| -------------------------- | -------------------------------------------------------------- |
| `main` (standaard)          | Alle privéberichten delen de [hoofdsessie](/concepts/main-session) |
| `per-peer`                 | Isoleren op afzender, over kanalen heen                        |
| `per-channel-peer`         | Isoleren op kanaal + afzender (aanbevolen)                     |
| `per-account-channel-peer` | Isoleren op account + kanaal + afzender                        |

<Tip>
Als dezelfde persoon via meerdere kanalen contact met je opneemt, gebruik dan
`session.identityLinks` om diens identiteiten aan één canonieke peer-id te koppelen,
zodat ze een sessie delen.
</Tip>

### Gekoppelde kanalen docken

Dockopdrachten verplaatsen de antwoordroute van de huidige sessie voor
privéchats naar een ander gekoppeld kanaal zonder een nieuwe sessie te starten.
Zie [Kanalen docken](/nl/concepts/channel-docking) voor voorbeelden, configuratie
en probleemoplossing.

Controleer je configuratie met `openclaw security audit`.

## Incognitosessies

Incognitosessies zijn alleen beschikbaar vanuit het scherm **Nieuwe thread** van de Control UI. Schakel **Incognito** in voordat je de thread start, zodat de sessievermelding, het transcript en de Compaction-status in het procesgeheugen worden bewaard in plaats van op schijf. De thread verdwijnt wanneer de Gateway opnieuw wordt gestart, voert de automatische geheugenflush van OpenClaw niet uit en maakt geen transcriptarchief wanneer je de thread reset of verwijdert. Uitvoeringen op basis van Codex starten hun harnessthread ook in tijdelijke modus, zodat Codex geen rollout- of lokale sessiestatusbestanden schrijft; andere modelproviders gebruiken HTTP-API's en bewaren geen lokaal providertranscript in OpenClaw.

Het segment `incognito-` is gereserveerd voor sessiesleutels van het dashboard, subagents en verborgen interne sessies; `openclaw doctor --fix` hernoemt conflicterende verouderde duurzame sleutels.

Incognito beperkt de normale tools van de agent niet. Een expliciet verzoek om informatie op te slaan of het schrijven van bestanden door een tool kan nog steeds gegevens buiten de incognitosessieopslag bewaren. Je geconfigureerde modelprovider verwerkt nog steeds de berichten die je verzendt, diagnostische logboekregistratie blijft ongewijzigd en OpenClaw registreert nog steeds inhoudsvrije auditmetadata, zoals HMAC-verwijzingen.

Op Gateways met meerdere gebruikers zijn incognitothreads alleen zichtbaar voor verbindingen met beheerdersbereik en verschijnen ze nooit via de agentsessietools of transcriptzoekfunctie van een andere sessie. Dit beschermt ze tegen opslag en andere gebruikers die via de Gateway werken, maar niet tegen de eigenaar van de Gateway of de procesbeheerder, die live sessies altijd kan observeren.

## Onthouden tussen gesprekken

Afzonderlijke transcripten bepalen de lokale geschiedenis van elk gesprek. Voor
een persoonlijke of volledig vertrouwde agent voegt `memory.search.rememberAcrossConversations: true`
een optionele ophaalstap toe voor de andere privégesprekken van die agent; de
transcripten worden hierdoor niet samengevoegd.

Privégesprekken en permanente expliciete UI-gesprekken kunnen elkaar relevante
context bieden. Groepen en kanalen blijven in beide richtingen gescheiden: hun
transcripten zijn geen bronnen voor het ophalen van privécontext en antwoorden
in die gesprekken ontvangen geen context uit privétranscripten. Het huidige
gesprek wordt ook uitgesloten, omdat de geschiedenis ervan al is geladen.

Deze instelling wijzigt geen sessiesleutels, bereik van privéberichten,
routering, bezorging of `tools.sessions.visibility`. Gedeeld werkruimtegeheugen in
`MEMORY.md` en `memory/*.md` behoudt eveneens het bestaande gedrag.
De huidige geheugenprovider moet beveiligd ophalen van privétranscripten
ondersteunen; contextengines zoals Lossless Claw blijven onafhankelijk en
kunnen ernaast worden uitgevoerd. Zie
[Active Memory](/nl/concepts/active-memory#remember-across-conversations) voor
configuratie- en runtimedetails.

## Levenscyclus van sessies

Sessies worden hergebruikt totdat je ze handmatig reset of kiest voor een
automatisch resetbeleid:

- **Geen automatische reset** (standaard `mode: "none"`) - sessies behouden dezelfde
  `sessionId`; Compaction beheert de actieve context naarmate het gesprek groeit.
- **Dagelijkse reset** (`mode: "daily"`) - kies voor een nieuwe sessie op een geconfigureerd lokaal
  uur (`session.reset.atHour`, standaard `4`, 0-23) op de Gateway-host. De
  dagelijkse actualiteit is gebaseerd op het moment waarop de huidige
  `sessionId` begon, niet op latere schrijfbewerkingen van metadata.
- **Reset bij inactiviteit** (`mode: "idle"`) - kies voor een nieuwe sessie na
  `session.reset.idleMinutes` inactiviteit. De actualiteit voor inactiviteit is gebaseerd
  op de laatste echte gebruikers-/kanaalinteractie, zodat Heartbeat-, Cron- en
  exec-systeemgebeurtenissen de sessie niet actief houden.
- **Handmatige reset** - typ `/new` of `/reset` in de chat. `/new <model>` wisselt ook
  van model.

Wanneer zowel dagelijkse resets als resets bij inactiviteit zijn geconfigureerd,
geldt degene die het eerst verloopt. Heartbeat-, Cron-, exec- en andere
systeemgebeurtenisbeurten kunnen sessiemetadata schrijven, maar die
schrijfbewerkingen verlengen de actualiteit voor dagelijkse resets of resets
bij inactiviteit niet. Wanneer een reset de sessie vernieuwt, worden
systeemgebeurtenismeldingen in de wachtrij voor de oude sessie verwijderd,
zodat verouderde achtergrondupdates niet vóór de eerste prompt in de nieuwe
sessie worden geplaatst.

Sessies met een actieve CLI-sessie die eigendom is van de provider volgen
dezelfde standaard zonder automatische reset. Gebruik `/reset` of
configureer `session.reset` expliciet wanneer die sessies volgens een timer
moeten verlopen.

Schakel automatische resets globaal in en overschrijf ze vervolgens per
chattype of kanaal:

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType` ondersteunt `direct`, `group` en `thread`. Doctor migreert verouderde `dm`-vermeldingen naar `direct` en `session.idleMinutes` naar `session.reset.idleMinutes`; het schema weigert beide uitgefaseerde vormen.

## Waar de status wordt bewaard

- **Rijen met runtimesessies:** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Gearchiveerde transcriptbestanden:** `~/.openclaw/agents/<agentId>/sessions/`
- **Bron voor migratie van verouderde rijen:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`

De sessierijen in de SQLite-database per agent bewaren afzonderlijke
tijdstempels voor de levenscyclus:

- `sessionStartedAt`: wanneer de huidige `sessionId` begon; de dagelijkse reset gebruikt dit.
- `lastInteractionAt`: laatste gebruikers-/kanaalinteractie die de levensduur bij inactiviteit verlengt.
- `updatedAt`: laatste wijziging van de opslagrij; nuttig voor weergave en opschoning, maar niet
  bepalend voor de actualiteit van dagelijkse resets of resets bij inactiviteit.

Tijdens de migratie vanuit oudere installaties importeren het opstarten van de
Gateway en `openclaw doctor
--fix` automatisch verouderde `sessions.json`-rijen en
actieve transcriptgeschiedenis in JSONL-indeling naar SQLite. Rijen zonder
`sessionStartedAt` worden waar mogelijk herleid uit de sessieheader van het
verouderde JSONL-transcript. Als een oudere rij ook geen
`lastInteractionAt` bevat, valt de actualiteit voor inactiviteit terug op de
starttijd van die sessie, niet op latere administratieve schrijfbewerkingen.
Gebruik `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` en de
[Doctor-migratievolgorde](/nl/cli/doctor#session-sqlite-migration) wanneer je
expliciete inspectie of validatiebewijs wilt.

## Sessieonderhoud

OpenClaw beperkt de sessieopslag na verloop van tijd via
`session.maintenance`; de standaardwaarden worden hieronder weergegeven:

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" voert opschoning uit; "warn" rapporteert alleen
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

Voor `maxEntries`-limieten op productieschaal gebruiken
runtimeschrijfbewerkingen van de Gateway een kleine hoogwaterbuffer en wordt de
opslag in batches teruggebracht tot de geconfigureerde limiet. Bij het starten
van de Gateway worden tijdens leesbewerkingen van de sessieopslag geen
vermeldingen opgeschoond of begrensd, zodat het opstarten en geïsoleerde
Cron-sessies niet de kosten van een volledige opschoning van de opslag dragen.
`openclaw sessions cleanup --enforce` past de limiet onmiddellijk toe.

Probesessies voor modeluitvoeringen van de Gateway zijn standaard van korte
duur. Rijen die overeenkomen met `agent:*:explicit:model-run-<uuid>` gebruiken een vaste
bewaartermijn van `24h`, maar de opschoning is afhankelijk van
druk: verouderde proberijen worden alleen verwijderd wanneer de
onderhouds-/limietdruk voor sessievermeldingen wordt bereikt. Dit gebeurt vóór
de bredere leeftijdsgrens voor verouderde vermeldingen en vóór de limiet voor
het aantal vermeldingen. Normale sessies voor privéberichten, groepen,
threads, Cron, hooks, Heartbeat, ACP en subagents nemen deze bewaartermijn van
24h niet over.

Onderhoud behoudt duurzame externe gespreksverwijzingen, waaronder
groepssessies en chatsessies met threadbereik, terwijl synthetische vermeldingen
voor Cron, hooks, Heartbeat, ACP en subagents nog steeds kunnen verouderen.

Gearchiveerde sessies zijn door de gebruiker opgeborgen en vrijgesteld van elk
automatisch onderhoudspad, waaronder opschoning op basis van leeftijd,
vermeldingslimieten, opschoning van modeluitvoeringen en verwijdering vanwege
het schijfbudget. Ze blijven gearchiveerd totdat je ze uit het archief haalt of
expliciet verwijdert.

Als je eerder isolatie van privéberichten gebruikte en
`session.dmScope` later terugzette naar `main`, bekijk dan een
voorbeeld van verouderde, op peersleutels gebaseerde rijen voor privéberichten
met `openclaw sessions cleanup --dry-run --fix-dm-scope`. Door dezelfde vlag toe te passen,
worden die oude rijen voor directe privéberichten uitgefaseerd en blijven hun
transcripten als verwijderde archieven bewaard.

Bekijk een voorbeeld van elke onderhoudsuitvoering met `openclaw sessions cleanup --dry-run`.

## Sessies inspecteren

| Opdracht                    | Toont                                                   |
| --------------------------- | ------------------------------------------------------- |
| `openclaw status`          | Pad naar de sessieopslag en recente activiteit          |
| `openclaw sessions --json` | Alle sessies (filter met `--active <minutes>`) |
| `/status` in de chat       | Contextgebruik, model en schakelaars                    |
| `/context list`            | Wat de systeemprompt bevat                              |

## Verder lezen

- [Sessies doorzoeken](/nl/concepts/session-search) - zoeken in volledige tekst van eerdere transcripten
- [Sessies opschonen](/nl/concepts/session-pruning) - toolresultaten inkorten
- [Compaction](/nl/concepts/compaction) - lange gesprekken samenvatten
- [Sessietools](/nl/concepts/session-tool) - agenttools voor sessieoverschrijdend werk
- [Diepgaande uitleg over sessiebeheer](/nl/reference/session-management-compaction) -
  opslagschema, transcripten, verzendbeleid, oorsprongsmetadata en geavanceerde configuratie
- [Multi-agent](/nl/concepts/multi-agent) - routering en sessie-isolatie tussen agents
- [Achtergrondtaken](/nl/automation/tasks) - hoe losgekoppeld werk taakrecords met sessieverwijzingen maakt
- [Kanaalroutering](/nl/channels/channel-routing) - hoe inkomende berichten naar sessies worden gerouteerd

## Gerelateerd

- [Sessies opschonen](/nl/concepts/session-pruning)
- [Sessietools](/nl/concepts/session-tool)
- [Opdrachtwachtrij](/nl/concepts/queue)
