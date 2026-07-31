---
read_when:
    - Je wilt achtergrondwerk of parallel werk via de agent
    - Je wijzigt het beleid voor sessions_spawn of de sub-agenttool
    - Je implementeert of lost problemen op met threadgebonden subagentsessies
sidebarTitle: Sub-agents
summary: Start geïsoleerde agentruns op de achtergrond die de resultaten terugmelden in de chat van de aanvrager
title: Subagenten
x-i18n:
    generated_at: "2026-07-27T06:17:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e45b32fdb177c52ed785287712b9b6c2c30bbe392f0ce975970910ff91ed30ed
    source_path: tools/subagents.md
    workflow: 16
---

Sub-agents zijn agentruns op de achtergrond die vanuit een bestaande agentrun worden gestart.
Elk daarvan wordt uitgevoerd in een eigen sessie (`agent:<agentId>:subagent:<uuid>`) en
**kondigt** na voltooiing het resultaat aan in het chatkanaal van de aanvrager.
Elke sub-agentrun wordt bijgehouden als een [achtergrondtaak](/nl/automation/tasks).

Doelen:

- Onderzoek, langdurige taken en traag toolwerk parallel uitvoeren zonder de hoofdrun te blokkeren.
- Sub-agents standaard geïsoleerd houden (gescheiden sessies, optionele sandboxing).
- Zorgen dat het tooloppervlak moeilijk verkeerd te gebruiken is: sub-agents krijgen standaard **geen** sessie- of berichtentools.
- Configureerbare nestingsdiepte voor orchestratorpatronen ondersteunen.

<Note>
**Opmerking over kosten:** elke sub-agent heeft standaard een eigen context en
tokengebruik. Stel voor zware of repetitieve taken een goedkoper model voor sub-agents
in en houd je hoofdagent op een model van hogere kwaliteit via
`agents.defaults.subagents.model` of overrides per agent. Wanneer een onderliggende agent
het huidige transcript van de aanvrager echt nodig heeft, start je deze met
`context: "fork"`. Aan threads gebonden sub-agentsessies gebruiken standaard
`context: "fork"`, omdat ze het huidige gesprek vertakken naar een
vervolgthread.
</Note>

## Slash-opdracht

`/subagents` inspecteert sub-agentruns voor de **huidige sessie**:

```text
/subagents list
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
```

`/subagents info` toont metadata van de run (status, tijdstempels, sessie-id,
transcriptpad, opschoning). `/subagents log` toont recente chatbeurten voor een
run; voeg het token `tools` toe om berichten met toolaanroepen/-resultaten op te nemen (standaard
weggelaten). Gebruik `sessions_history` voor een begrensde, op veiligheid gefilterde
weergave om vanuit een agentbeurt informatie terug te halen, of inspecteer het transcriptpad op schijf voor
het onbewerkte volledige transcript.

In de Control UI hebben bovenliggende sessies met recente onderliggende runs een uitvouwbare
zijbalkrij. De geneste rijen tonen de status en looptijd van onderliggende runs, en als je er een selecteert,
wordt de chat van die onderliggende run geopend met behoud van de bovenliggende hiërarchie.

### Besturingselementen voor threadbinding

Deze opdrachten werken op kanalen met permanente threadbindingen. Zie
[Kanalen die threads ondersteunen](#thread-supporting-channels) hieronder.

```text
/focus <subagent-label|session-key|session-id|session-label>
/unfocus
/agents
/session idle <duration|off>
/session max-age <duration|off>
```

### Startgedrag

Agents starten sub-agents op de achtergrond met de tool `sessions_spawn`.
Voltooiingen worden teruggestuurd als interne gebeurtenissen van de bovenliggende sessie; de bovenliggende/aanvragende
agent bepaalt of een gebruikersgerichte update nodig is.

<AccordionGroup>
  <Accordion title="Niet-blokkerende, pushgebaseerde voltooiing">
    - `sessions_spawn` is niet-blokkerend; deze retourneert onmiddellijk een run-id.
    - Na voltooiing rapporteert de sub-agent terug aan de bovenliggende/aanvragende sessie.
    - Agentbeurten die resultaten van onderliggende agents nodig hebben, moeten na het starten van het vereiste werk `sessions_yield` aanroepen. Dit beëindigt de huidige beurt en zorgt ervoor dat de voltooiingsgebeurtenis als het volgende voor het model zichtbare bericht binnenkomt.
    - Voltooiing is pushgebaseerd. Poll na het starten **niet** herhaaldelijk `/subagents list`, `sessions_list` of `sessions_history` om alleen maar te wachten tot de run klaar is; controleer de status uitsluitend op aanvraag bij foutopsporing.
    - De uitvoer van een onderliggende agent is een rapport/bewijsstuk dat de aanvragende agent moet verwerken. Het is geen door de gebruiker geschreven instructietekst en kan systeem-, ontwikkelaars- of gebruikersbeleid niet overschrijven.
    - Na voltooiing sluit OpenClaw naar beste vermogen bijgehouden browsertabbladen/-processen die door die sub-agentsessie zijn geopend voordat de opschoningsflow van de aankondiging verdergaat.

  </Accordion>
  <Accordion title="Levering van voltooiingen">
    - OpenClaw stuurt voltooiingen terug naar de aanvragende sessie via een `agent`-beurt met een stabiele idempotentiesleutel.
    - Als de aanvragende run nog actief is, probeert OpenClaw die run eerst te activeren/bijsturen in plaats van een tweede zichtbaar antwoordpad te starten.
    - Als een actieve aanvrager niet kan worden geactiveerd, valt OpenClaw terug op een overdracht aan de aanvragende agent met dezelfde voltooiingscontext in plaats van de aankondiging te laten vervallen.
    - Een geslaagde overdracht aan de bovenliggende agent voltooit de levering van de sub-agent, zelfs als de bovenliggende agent besluit dat geen zichtbare gebruikersupdate nodig is.
    - Native sub-agents krijgen de berichtentool niet. Ze retourneren platte assistenttekst aan de bovenliggende/aanvragende agent; voor mensen zichtbare antwoorden blijven onder het normale leveringsbeleid van de bovenliggende/aanvragende agent vallen.
    - Als rechtstreekse overdracht niet kan worden gebruikt, valt de levering terug op routering via de wachtrij en vervolgens op een korte poging om de aankondiging opnieuw te leveren met exponentiële back-off, voordat definitief wordt opgegeven.
    - De levering behoudt de opgeloste route van de aanvrager: voltooiingsroutes die aan een thread of gesprek zijn gebonden, krijgen voorrang wanneer ze beschikbaar zijn. Als de oorsprong van de voltooiing alleen een kanaal opgeeft, vult OpenClaw het ontbrekende doel/account aan vanuit de opgeloste route van de aanvragende sessie (`lastChannel` / `lastTo` / `lastAccountId`), zodat rechtstreekse levering nog steeds werkt.

  </Accordion>
  <Accordion title="Metadata voor overdracht van voltooiingen">
    De overdracht van de voltooiing aan de aanvragende sessie is tijdens runtime gegenereerde
    interne context (geen door de gebruiker geschreven tekst) en bevat:

    - `Result` — de meest recente zichtbare `assistant`-antwoordtekst van de onderliggende agent. Tool-/toolResult-uitvoer wordt niet opgenomen in resultaten van onderliggende agents. Runs die terminaal zijn mislukt, gebruiken vastgelegde antwoordtekst niet opnieuw.
    - `Status` — `completed; ready for parent review` / `failed` / `timed out` / `unknown`.
    - Compacte runtime-/tokenstatistieken.
    - Een controle-instructie die de aanvragende agent opdraagt het resultaat te verifiëren voordat deze beslist of de oorspronkelijke taak is voltooid.
    - Vervolgrichtlijnen die de aanvragende agent opdragen de taak voort te zetten of een vervolgactie vast te leggen wanneer het resultaat van de onderliggende agent meer actie vereist.
    - Een instructie voor de laatste update voor het pad waarop geen verdere actie nodig is, geschreven in de normale assistentstijl zonder onbewerkte interne metadata door te sturen.

  </Accordion>
  <Accordion title="Modi en ACP-runtime">
    - `--model` en `--thinking` overschrijven de standaardwaarden voor die specifieke run.
    - Gebruik `info`/`log` om na voltooiing details en uitvoer te inspecteren.
    - Gebruik voor permanente aan threads gebonden sessies `sessions_spawn` met `thread: true` en `mode: "session"`.
    - Als het kanaal van de aanvrager geen threadbindingen ondersteunt, gebruik je `mode: "run"` in plaats van een onmogelijke aan threads gebonden combinatie opnieuw te proberen.
    - Gebruik voor ACP-harnesssessies (Claude Code, Gemini CLI, OpenCode of expliciete Codex ACP/acpx) `sessions_spawn` met `runtime: "acp"` wanneer de tool die runtime bekendmaakt. Zie [ACP-leveringsmodel](/nl/tools/acp-agents#delivery-model) bij het opsporen van fouten in voltooiingen of lussen tussen agents. Wanneer de Plugin `codex` is ingeschakeld, moet de besturing van Codex-chats/-threads de voorkeur geven aan `/codex ...` boven ACP, tenzij de gebruiker expliciet om ACP/acpx vraagt.
    - OpenClaw verbergt `runtime: "acp"` totdat ACP is ingeschakeld, de aanvrager niet in een sandbox draait en een backend-Plugin zoals `acpx` is geladen. `runtime: "acp"` verwacht een externe ACP-harness-id, of een `agents.entries.*`-item met `runtime.type="acp"`; gebruik de standaardruntime voor sub-agents voor normale OpenClaw-configuratieagents uit `agents_list`.

  </Accordion>
</AccordionGroup>

## Contextmodi

Native sub-agents starten geïsoleerd, tenzij de aanroeper expliciet vraagt om het
huidige transcript te vertakken.

| Modus       | Wanneer te gebruiken                                                                                                                         | Gedrag                                                                          |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `isolated` | Nieuw onderzoek, onafhankelijke implementatie, traag toolwerk of alles wat in de taaktekst kan worden toegelicht                           | Maakt een schoon transcript voor de onderliggende agent. Dit is de standaard en houdt het tokengebruik lager.  |
| `fork`     | Werk dat afhankelijk is van het huidige gesprek, eerdere toolresultaten of genuanceerde instructies die al in het transcript van de aanvrager staan | Vertakt het transcript van de aanvrager naar de onderliggende sessie voordat de onderliggende agent start. |

Gebruik `fork` spaarzaam. Het is bedoeld voor contextgevoelige delegatie, niet als
vervanging voor het schrijven van een duidelijke taakprompt.

## Tool: `sessions_spawn`

Start een sub-agentrun met `deliver: false` op de algemene `subagent`-lane,
voert vervolgens een aankondigingsstap uit en plaatst het aankondigingsantwoord in het
chatkanaal van de aanvrager.

De beschikbaarheid hangt af van het effectieve toolbeleid van de aanroeper. De ingebouwde
profielen `coding` en `messaging` bevatten `sessions_spawn`,
`sessions_yield` en `subagents`; `minimal` bevat deze niet. `full` staat elke
tool toe. Voeg deze tools toe met `tools.alsoAllow`, of gebruik een van de bovenstaande
profielen, voor een agent met een aangepast beperkter profiel die toch
werk moet kunnen delegeren.
Beleid voor kanalen/groepen, providers, sandboxen en toestaan/weigeren per agent kan
de tool na de profielfase alsnog verwijderen. Gebruik `/tools` vanuit dezelfde
sessie om de effectieve toollijst te bevestigen.

**Standaardwaarden:**

- **Model:** native sub-agents nemen het model van de aanroeper over, tenzij je `agents.defaults.subagents.model` instelt (of `agents.entries.*.subagents.model` per agent). Starts via de ACP-runtime gebruiken hetzelfde geconfigureerde sub-agentmodel wanneer dit aanwezig is; anders behoudt de ACP-harness zijn eigen standaardmodel. Een expliciete `sessions_spawn.model` krijgt nog steeds voorrang.
- **Redeneren:** native sub-agents nemen de redeneerinstelling van de aanroeper over, tenzij je `agents.defaults.subagents.thinking` instelt (of `agents.entries.*.subagents.thinking` per agent). Starts via de ACP-runtime passen ook `agents.defaults.models["provider/model"].params.thinking` toe voor het geselecteerde model. Een expliciete `sessions_spawn.thinking` krijgt nog steeds voorrang.
- **Time-out van run:** OpenClaw gebruikt `agents.defaults.subagents.runTimeoutSeconds` wanneer dit is ingesteld; anders valt het terug op `0` (geen time-out). `sessions_spawn` accepteert geen time-outoverschrijvingen per aanroep.
- **Proceslevensduur:** een losgekoppelde OpenClaw-sub-agent heeft een eigen runlevenscyclus. Een achtergrondtaak die binnen een externe CLI-backend wordt gemaakt, is anders: deze deelt het bovenliggende CLI-subproces en stopt als dat bovenliggende proces `agents.defaults.timeoutSeconds` bereikt.
- **Taaklevering:** native sub-agents ontvangen de gedelegeerde taak in hun eerste zichtbare `[Subagent Task]`-bericht. De systeemprompt van de sub-agent bevat runtimeregels en routeringscontext, niet een verborgen duplicaat van de taak.

Geaccepteerde starts van native sub-agents bevatten de opgeloste modelmetadata van de onderliggende agent
in het toolresultaat: `resolvedModel` bevat de toegepaste modelreferentie en
`resolvedProvider` bevat het providerprefix wanneer de referentie er een heeft.

### Modus voor delegatieprompts

`agents.defaults.subagents.delegationMode` bepaalt alleen de promptrichtlijnen; deze instelling wijzigt het toolbeleid niet en dwingt delegatie niet af.

- `suggest` (standaard): behoud de standaardaanmoediging in de prompt om sub-agents te gebruiken voor groter of trager werk.
- `prefer`: draag de hoofdagent op responsief te blijven en alles wat uitgebreider is dan een rechtstreeks antwoord via `sessions_spawn` te delegeren.

Override per agent: `agents.entries.*.subagents.delegationMode`.

```json5
{
  agents: {
    defaults: {
      subagents: {
        delegationMode: "prefer",
        maxConcurrent: 4,
      },
    },
    list: [
      {
        id: "coordinator",
        subagents: { delegationMode: "prefer" },
      },
    ],
  },
}
```

### Toolparameters

<ParamField path="task" type="string" required>
  De taakbeschrijving voor de subagent.
</ParamField>
<ParamField path="taskName" type="string">
  Optionele stabiele aanduiding om een specifiek kind later in statusuitvoer te identificeren. Moet overeenkomen met `[a-z][a-z0-9_-]{0,63}` en mag geen gereserveerd doel zijn, zoals `last` of `all`.
</ParamField>
<ParamField path="label" type="string">
  Optioneel, voor mensen leesbaar label.
</ParamField>
<ParamField path="agentId" type="string">
  Start onder een andere geconfigureerde agent-id wanneer dit is toegestaan door `subagents.allowAgents`.
</ParamField>
<ParamField path="cwd" type="string">
  Optionele werkmap voor de taak van de kindrun. Native subagents laden bootstrapbestanden nog steeds vanuit de werkruimte van de doelagent; `cwd` verandert alleen waar runtimehulpmiddelen en CLI-harnassen het gedelegeerde werk uitvoeren.
</ParamField>
<ParamField path="runtime" type='"subagent" | "acp"' default="subagent">
  `acp` is alleen bedoeld voor externe ACP-harnassen (`claude`, `droid`, `gemini`, `opencode` of expliciet aangevraagde Codex ACP/acpx) en voor `agents.entries.*`-items waarvan `runtime.type` gelijk is aan `acp`.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Alleen voor ACP. Hervat een bestaande ACP-harnassessie wanneer `runtime: "acp"`; wordt genegeerd voor het starten van native subagents.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  Alleen voor ACP. Streamt ACP-runuitvoer naar de bovenliggende sessie wanneer `runtime: "acp"`; weglaten bij het starten van native subagents.
</ParamField>
<ParamField path="model" type="string">
  Overschrijf het model van de subagent. Ongeldige waarden worden overgeslagen en de subagent draait op het standaardmodel, met een waarschuwing in het hulpmiddelresultaat.
</ParamField>
<ParamField path="thinking" type="string">
  Overschrijf het denkniveau voor de subagentrun. Niet beschikbaar met `visible: true`.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Wanneer `true`, wordt kanaalthreadkoppeling aangevraagd voor deze subagentsessie.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  Als `thread: true` en `mode` is weggelaten, wordt de standaardwaarde `session`. `mode: "session"` vereist `thread: true`.
  Als threadkoppeling niet beschikbaar is voor het kanaal van de aanvrager, gebruik dan in plaats daarvan `mode: "run"`.
  Laat bij `visible: true` `mode` weg; zichtbare sessies zijn persistent en ondersteunen `mode: "run"` niet.
</ParamField>
<ParamField path="cleanup" type='"delete" | "keep"' default="keep">
  `"delete"` archiveert de sessie onmiddellijk na de aankondiging (het transcript blijft via hernoemen behouden).
</ParamField>
<ParamField path="sandbox" type='"inherit" | "require"' default="inherit">
  `require` weigert het starten tenzij de runtime van het doelkind in een sandbox draait.
</ParamField>
<ParamField path="context" type='"isolated" | "fork"' default="isolated">
  `fork` vertakt het huidige transcript van de aanvrager naar de kindsessie. Alleen native subagents. Aan threads gekoppelde starts gebruiken standaard `fork`; niet aan threads gekoppelde starts gebruiken standaard `isolated`. Een zichtbare vertakking moet dezelfde agent als de aanvrager als doel hebben.
</ParamField>
<ParamField path="visible" type="boolean" default="false">
  Maak een persistente dashboardsessie die de gebruiker in de Control UI kan openen. Zichtbare starts ondersteunen alleen `runtime: "subagent"` en behouden de gemaakte sessie altijd.
</ParamField>
<ParamField path="worktree" type="boolean" default="false">
  Richt een beheerde git-worktree in voor de nieuwe dashboardsessie. Vereist `visible: true`.
</ParamField>
<ParamField path="worktreeName" type="string">
  Optionele naam voor de beheerde worktree. Vereist `visible: true` en `worktree: true`.
</ParamField>
<ParamField path="worktreeBaseRef" type="string">
  Optionele git-basisreferentie voor de beheerde worktree. Vereist `visible: true` en `worktree: true`.
</ParamField>

<Warning>
`sessions_spawn` accepteert **geen** parameters voor kanaalbezorging (`target`,
`channel`, `to`, `threadId`, `replyTo`, `transport`). Native subagents melden
hun laatste assistentbeurt terug aan de aanvrager; externe bezorging blijft bij
de bovenliggende/aanvragende agent.
</Warning>

Met `visible: true` worden `model`, `cwd` en een `context: "fork"` voor dezelfde agent ondersteund. Een doel in een sandbox beperkt `cwd` tot de werkruimte van die agent. Threadkoppeling, `mode`, overschrijvingen van het denkniveau, `lightContext`, `attachments` en `attachAs` zijn op dit pad niet beschikbaar, omdat zichtbare sessies persistente dashboardsessies zijn die via `sessions.create` worden gemaakt. Zichtbaar starten wordt geweigerd wanneer de aanvrager zelf is gestart met een overgenomen toestemmings- of weigeringslijst voor hulpmiddelen; die beperking wordt bij het starten vastgelegd en kan niet via configuratie worden overschreven. Het weergeven en adresseren van sessies volgt `tools.sessions.visibility`; het standaardbereik `tree` omvat de huidige sessie en de eigen onderliggende startstructuur. Zie [Beheerde worktrees](/nl/concepts/managed-worktrees) voor het benoemen van check-outs en voor het gedrag bij installatie, opschoning en herstel.

### Taaknamen en doelen

`taskName` is een modelgerichte aanduiding voor orkestratie, geen sessiesleutel.
Gebruik deze voor stabiele kindnamen zoals `review_subagents`,
`linux_validation` of `docs_update` wanneer een coördinator dat kind later
mogelijk moet inspecteren.

Bij doelresolutie worden exacte overeenkomsten met `taskName` en eenduidige
voorvoegsels geaccepteerd. Overeenkomsten zijn beperkt tot hetzelfde venster met actieve/recente doelen dat
wordt gebruikt door genummerde `/subagents`-doelen, zodat een verouderd voltooid kind een
hergebruikte aanduiding niet dubbelzinnig maakt. Als twee actieve of recente kinderen dezelfde
`taskName` delen, is het doel dubbelzinnig; gebruik in plaats daarvan de lijstindex, sessiesleutel of
run-id.

De gereserveerde doelen `last` en `all` zijn geen geldige waarden voor `taskName`,
omdat ze al een besturingsbetekenis hebben.

## Hulpmiddel: `sessions_yield`

Beëindigt de huidige modelbeurt en wacht tot runtimegebeurtenissen, voornamelijk
voltooiingsgebeurtenissen van subagents, als het volgende bericht binnenkomen. Gebruik dit na
het starten van vereist kinderwerk wanneer de aanvrager geen definitief
antwoord kan geven voordat die voltooiingen zijn binnengekomen.

`sessions_yield` is het wachtmechanisme. Vervang het niet door pollinglussen
over `subagents`, `sessions_list`, `sessions_history`, shell-
`sleep` of procespolling alleen om te detecteren dat een kind is voltooid.

Gebruik `sessions_yield` alleen wanneer de effectieve hulpmiddelenlijst van de sessie
dit bevat. Sommige minimale of aangepaste hulpmiddelprofielen kunnen `sessions_spawn` en
`subagents` beschikbaar stellen zonder `sessions_yield`; verzin in dat geval geen
pollinglus alleen om op voltooiing te wachten.

Wanneer er actieve kinderen zijn, voegt OpenClaw een compact, door de runtime gegenereerd
`Active Subagents`-promptblok in normale beurten in, zodat de aanvrager
de huidige kindsessies, run-id's, statussen, labels, taken en
`taskName`-aliassen kan zien zonder polling. De taak- en labelvelden in dat
blok worden als gegevens aangehaald, niet als instructies, omdat ze afkomstig kunnen zijn
van door gebruikers of modellen verstrekte startargumenten.

## Hulpmiddel: `subagents`

Toont gestarte subagentruns en achtergrondtaakrecords die eigendom zijn van de
sessiestructuur van de aanvrager. De taakrijen omvatten native subagents, ACP-runs,
Gateway-CLI-/mediawerk en Cron-uitvoeringen. Het bereik is beperkt tot de huidige
aanvrager; een kind kan alleen zijn eigen beheerde kinderen zien.

Gebruik `subagents` voor status op aanvraag en foutopsporing. Gebruik `sessions_yield` om
op voltooiingsgebeurtenissen te wachten.

Gebruik `action: "cancel"` met een door `action: "list"` geretourneerde `taskId` om
een taak te stoppen. Annulering is beperkt tot de beheerde sessiestructuur; een subagent
zonder kinderen kan geen werk annuleren dat eigendom is van een andere sessie.

## Aan threads gekoppelde sessies

Wanneer threadkoppelingen voor een kanaal zijn ingeschakeld, kan een subagent aan
een thread gekoppeld blijven, zodat volgende gebruikersberichten in die thread naar
dezelfde subagentsessie blijven worden gerouteerd.

### Kanalen die threads ondersteunen

Een kanaal ondersteunt persistente, aan threads gekoppelde subagentsessies
(`sessions_spawn` met `thread: true`) wanneer het een adapter voor gesprekskoppeling
registreert. Meegeleverde kanalen met deze ondersteuning: **Discord**,
**iMessage**, **Matrix** en **Telegram**. Discord en Matrix maken standaard
een kindthread; Telegram en iMessage koppelen standaard het
huidige gesprek. Gebruik de kanaalspecifieke `threadBindings`-configuratiesleutels voor
inschakeling, time-outs en `spawnSessions`.

### Snelle procedure

<Steps>
  <Step title="Starten">
    `sessions_spawn` met `thread: true` (en optioneel `mode: "session"`).
  </Step>
  <Step title="Koppelen">
    OpenClaw maakt of koppelt in het actieve kanaal een thread aan dat sessiedoel.
  </Step>
  <Step title="Vervolgberichten routeren">
    Antwoorden en vervolgberichten in die thread worden naar de gekoppelde sessie gerouteerd.
  </Step>
  <Step title="Time-outs inspecteren">
    Gebruik `/session idle` om het automatisch opheffen van de focus na inactiviteit te inspecteren/bij te werken en
    `/session max-age` om de harde limiet te beheren.
  </Step>
  <Step title="Ontkoppelen">
    Gebruik `/unfocus` om handmatig te ontkoppelen.
  </Step>
</Steps>

### Handmatige bediening

| Opdracht            | Effect                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `/focus <target>`  | Koppel de huidige thread (of maak er een) aan een subagent-/sessiedoel                     |
| `/unfocus`         | Verwijder de koppeling voor de huidige gekoppelde thread                                           |
| `/agents`          | Toon actieve runs en koppelingsstatus (`binding:<id>`, `unbound` of `bindings unavailable`) |
| `/session idle`    | Inspecteer/werk automatisch opheffen van de focus bij inactiviteit bij (alleen gekoppelde threads met focus)                             |
| `/session max-age` | Inspecteer/werk de harde limiet bij (alleen gekoppelde threads met focus)                                      |

### Configuratieschakelaars

- **Algemene standaardwaarde:** `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
- **Kanaaloverschrijving en sleutels voor automatisch koppelen bij starten** zijn adapterspecifiek. Zie [Kanalen die threads ondersteunen](#thread-supporting-channels) hierboven.

Zie [Configuratiereferentie](/nl/gateway/configuration-reference) en
[Slash-opdrachten](/nl/tools/slash-commands) voor actuele adapterdetails.

### Toestemmingslijst

<ParamField path="agents.entries.*.subagents.allowAgents" type="string[]">
  Lijst met geconfigureerde agent-id's die via expliciete `agentId` als doel kunnen worden gekozen (`["*"]` staat elk geconfigureerd doel toe). Standaard: alleen de aanvragende agent. Als je een lijst instelt en nog steeds wilt dat de aanvrager zichzelf met `agentId` start, neem dan de id van de aanvrager in de lijst op.
</ParamField>
<ParamField path="agents.defaults.subagents.allowAgents" type="string[]">
  Standaardtoestemmingslijst voor geconfigureerde doelagents die wordt gebruikt wanneer de aanvragende agent geen eigen `subagents.allowAgents` instelt.
</ParamField>
<ParamField path="agents.defaults.subagents.requireAgentId" type="boolean" default="false">
  Blokkeer `sessions_spawn`-aanroepen die `agentId` weglaten (dwingt expliciete profielselectie af). Overschrijving per agent: `agents.entries.*.subagents.requireAgentId`.
</ParamField>
<ParamField path="agents.defaults.subagents.announceTimeoutMs" type="number" default="120000">
  Time-out per aanroep voor bezorgpogingen van Gateway-`agent`-aankondigingen. Waarden zijn positieve gehele aantallen milliseconden en worden begrensd op het platformveilige timermaximum. Tijdelijke nieuwe pogingen kunnen de totale wachttijd voor aankondigingen langer maken dan één geconfigureerde time-out.
</ParamField>

Als de aanvragende sessie in een sandbox draait, weigert `sessions_spawn` doelen
die zonder sandbox zouden worden uitgevoerd.

### Detectie

Gebruik `agents_list` om te zien welke agent-id's momenteel zijn toegestaan voor
`sessions_spawn`. Het antwoord bevat voor elke vermelde agent het effectieve
model en ingesloten runtime-metadata, zodat aanroepers onderscheid kunnen maken tussen OpenClaw, de Codex
app-server en andere geconfigureerde native runtimes.

`allowAgents`-vermeldingen moeten verwijzen naar geconfigureerde agent-id's in `agents.entries.*`.
`["*"]` betekent elke geconfigureerde doelagent plus de aanvrager. Als een agentconfiguratie
wordt verwijderd maar het id ervan in `allowAgents` blijft staan, weigert `sessions_spawn` dat id
en laat `agents_list` het weg. Voer `openclaw doctor --fix` uit om verouderde
toelijstvermeldingen op te schonen, of voeg een minimale `agents.entries.*`-vermelding toe wanneer het doel
startbaar moet blijven en daarbij standaardwaarden moet overnemen.

### Automatisch archiveren

- Subagentsessies worden automatisch gearchiveerd na `agents.defaults.subagents.archiveAfterMinutes` (standaard `60`).
- Voor archivering wordt `sessions.delete` gebruikt en het transcript wordt hernoemd naar `*.deleted.<timestamp>` (dezelfde map).
- `cleanup: "delete"` archiveert direct na de aankondiging (het transcript blijft behouden via hernoemen).
- Automatisch archiveren gebeurt op basis van beste inspanning; timers die nog wachten gaan verloren als de Gateway opnieuw wordt gestart.
- Geconfigureerde uitvoeringstime-outs archiveren **niet** automatisch; ze stoppen alleen de uitvoering. De sessie blijft bestaan tot de automatische archivering.
- Automatisch archiveren geldt in gelijke mate voor sessies op diepte 1 en diepte 2.
- Browseropschoning staat los van archiefopschoning: bijgehouden browsertabbladen/-processen worden op basis van beste inspanning gesloten wanneer de uitvoering eindigt, zelfs als het transcript/de sessieregistratie wordt bewaard.

## Geneste subagents

Standaard kunnen subagents geen eigen subagents starten
(`maxSpawnDepth: 1`). Stel `maxSpawnDepth: 2` in om één niveau
van nesting in te schakelen — het **orchestratorpatroon**: hoofdagent → orchestrator-subagent →
werker-sub-subagents.

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // subagents toestaan kinderen te starten (standaard: 1, bereik 1-5)
        maxChildrenPerAgent: 5, // max. actieve kinderen per agentsessie (standaard: 5, bereik 1-20)
        maxConcurrent: 8, // globale limiet voor gelijktijdigheidslane (standaard: 8)
        runTimeoutSeconds: 900, // standaardtime-out voor sessions_spawn (0 = geen time-out)
        announceTimeoutMs: 120000, // Gateway-time-out per aanroep voor aankondigingen
      },
    },
  },
}
```

### Diepteniveaus

| Diepte | Vorm van sessiesleutel                       | Rol                                           | Kan starten?                  |
| ----- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0     | `agent:<id>:main`                            | Hoofdagent                                    | Altijd                       |
| 1     | `agent:<id>:subagent:<uuid>`                 | Subagent (orchestrator wanneer diepte 2 is toegestaan) | Alleen als `maxSpawnDepth >= 2` |
| 2     | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | Sub-subagent (eindwerker)                     | Nooit                        |

### Aankondigingsketen

Resultaten stromen terug omhoog door de keten:

1. Werker op diepte 2 is klaar → kondigt dit aan bij zijn ouder (orchestrator op diepte 1).
2. Orchestrator op diepte 1 ontvangt de aankondiging, synthetiseert resultaten, is klaar → kondigt dit aan bij de hoofdagent.
3. Hoofdagent ontvangt de aankondiging en levert deze aan de gebruiker.

Elk niveau ziet alleen aankondigingen van zijn directe kinderen.

<Note>
**Operationele richtlijn:** start werk van kinderen eenmaal en wacht op voltooiingsgebeurtenissen
in plaats van pollinglussen te bouwen rond `sessions_list`,
`sessions_history`, `/subagents list` of `exec`-slaapopdrachten.
`sessions_list` en `/subagents list` houden relaties met kindsessies
gericht op actief werk — actieve kinderen blijven gekoppeld, beëindigde kinderen blijven
korte tijd zichtbaar in een recent venster en verouderde kindkoppelingen die alleen in de opslag bestaan,
worden na hun versheidsvenster genegeerd. Dit voorkomt dat oude `spawnedBy` /
`parentSessionKey`-metadata spookkinderen na een
herstart opnieuw tot leven wekken. Als een voltooiingsgebeurtenis van een kind binnenkomt nadat je het
definitieve antwoord al hebt verzonden, is de juiste vervolgstap exact het stille token
`NO_REPLY` / `no_reply`.
</Note>

### Toolbeleid per diepte

- Een kind legt bij het starten het effectieve afzenderbeleid van de aanvrager vast. Kinduitvoeringen zonder afzender en hervattingen door geverifieerde operators behouden die momentopname, zelfs als `toolsBySender` later verandert; huidige globale, agent-, provider-, sandbox- en subagentbeperkingen blijven van toepassing. Een nieuwe externe kanaalbeurt die op het kind is gericht, bepaalt in plaats daarvan opnieuw het huidige afzenderbeleid.
- Rol en besturingsbereik worden tijdens het starten in de sessiemetadata geschreven. Zo kunnen vlakke of herstelde sessiesleutels niet per ongeluk opnieuw orchestratorbevoegdheden krijgen.
- **Diepte 1 (orchestrator, wanneer `maxSpawnDepth >= 2`):** krijgt `sessions_spawn`, `subagents`, `sessions_list`, `sessions_history`, zodat deze kinderen kan starten en hun status kan inspecteren. Andere sessie-/systeemtools blijven geweigerd.
- **Diepte 1 (eindagent, wanneer `maxSpawnDepth == 1`):** geen sessietools (huidig standaardgedrag).
- **Diepte 2 (eindwerker):** geen sessietools — `sessions_spawn` wordt op diepte 2 altijd geweigerd. Kan geen verdere kinderen starten.

### Startlimiet per agent

Elke agentsessie (op elke diepte) kan tegelijkertijd maximaal `maxChildrenPerAgent`
(standaard `5`) actieve kinderen hebben. Dit voorkomt onbeheersbare uitwaaiering
vanuit één orchestrator.

### Cascadestop

Als een orchestrator op diepte 1 wordt gestopt, worden automatisch al zijn kinderen op diepte 2
gestopt:

- `/stop` in de hoofdchat stopt alle agents op diepte 1 en laat de stop doorwerken naar hun kinderen op diepte 2.

## Authenticatie

Authenticatie van subagents wordt bepaald op basis van **agent-id**, niet op basis van sessietype:

- De sessiesleutel van de subagent is `agent:<agentId>:subagent:<uuid>`.
- De authenticatieopslag wordt geladen vanuit `agentDir` van die agent.
- De authenticatieprofielen van de hoofdagent worden samengevoegd als **terugvaloptie**; agentprofielen overschrijven hoofdprofielen bij conflicten.

De samenvoeging is additief, waardoor hoofdprofielen altijd als
terugvaloptie beschikbaar zijn. Volledig geïsoleerde authenticatie per agent wordt nog niet ondersteund.

## Aankondiging

Subagents rapporteren terug via een aankondigingsstap:

- De aankondigingsstap wordt uitgevoerd binnen de subagentsessie (niet de sessie van de aanvrager).
- Als de subagent exact antwoordt met `ANNOUNCE_SKIP`, wordt er niets geplaatst.
- Als de nieuwste assistenttekst exact het stille token `NO_REPLY` / `no_reply` is, wordt de aankondigingsuitvoer onderdrukt, zelfs als er eerder zichtbare voortgang was.

De levering hangt af van de diepte van de aanvrager:

- Aanvragersessies op het hoogste niveau gebruiken een vervolgaanroep naar `agent` met externe levering (`deliver=true`).
- Geneste subagent-aanvragersessies ontvangen een interne vervolginjectie (`deliver=false`), zodat de orchestrator de resultaten van kinderen binnen de sessie kan synthetiseren.
- Als een geneste subagent-aanvragersessie verdwenen is, valt OpenClaw waar mogelijk terug op de aanvrager van die sessie.

Voor aanvragersessies op het hoogste niveau bepaalt directe levering in voltooiingsmodus eerst
een eventuele gekoppelde gespreks-/threadroute en hook-override, en vult vervolgens
ontbrekende kanaal-doelvelden aan vanuit de opgeslagen route van de aanvragersessie.
Daardoor blijven voltooiingen in de juiste chat/het juiste onderwerp, zelfs wanneer de oorsprong van de voltooiing
alleen het kanaal identificeert.

Aggregatie van kindvoltooiingen is bij het opbouwen van geneste voltooiingsbevindingen
beperkt tot de huidige uitvoering van de aanvrager, zodat verouderde kinduitvoer van een eerdere uitvoering
niet in de huidige aankondiging terechtkomt. Aankondigingsantwoorden behouden
thread-/onderwerproutering wanneer die beschikbaar is op kanaaladapters.

### Aankondigingscontext

De aankondigingscontext wordt genormaliseerd naar een stabiel intern gebeurtenisblok:

| Veld           | Bron                                                                                                     |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| Bron           | `subagent` of `cron`                                                                                     |
| Sessie-id's    | Sessiesleutel/-id van het kind                                                                           |
| Type           | Aankondigingstype + taaklabel                                                                            |
| Status         | Afgeleid van runtime-uitkomst (`ok`, `error`, `timeout` of `unknown`) — **niet** afgeleid uit modeltekst |
| Resultaatinhoud | Nieuwste zichtbare assistenttekst van het kind                                                           |
| Vervolgstap    | Instructie die beschrijft wanneer te antwoorden en wanneer stil te blijven                               |

Mislukte beëindigde uitvoeringen melden een foutstatus zonder vastgelegde
antwoordtekst opnieuw af te spelen. Tool-/toolResult-uitvoer wordt niet bevorderd tot resultaattekst van het kind.

### Statistiekenregel

Aankondigingspayloads bevatten aan het einde een statistiekenregel (zelfs bij regelterugloop):

- Uitvoeringstijd (bijv. `runtime 5m12s`).
- Tokengebruik (invoer/uitvoer/totaal).
- Geschatte kosten wanneer modelprijzen zijn geconfigureerd (`models.providers.*.models[].cost`).
- `sessionKey`, `sessionId` en het transcriptpad, zodat de hoofdagent de geschiedenis kan ophalen via `sessions_history` of het bestand op schijf kan inspecteren.

Interne metadata zijn alleen bedoeld voor orchestratie; gebruikersgerichte antwoorden
moeten worden herschreven in de normale assistentstem.

### Waarom `sessions_history` de voorkeur heeft

`sessions_history` is het veiligere orchestratiepad om het transcript van een kind
binnen een agentbeurt te lezen:

- Maskeert tekst die lijkt op inloggegevens/tokens, zelfs wanneer algemene logmaskering is uitgeschakeld.
- Kapt lange tekstblokken af (4000 tekens per blok) en verwijdert denksignaturen, payloads voor het opnieuw afspelen van redeneringen en inline afbeeldingsgegevens.
- Dwingt een antwoordlimiet van 80 KB af; te grote rijen worden vervangen door `[sessions_history omitted: message too large]`.
- Gebruik `nextOffset` indien aanwezig om achterwaarts door oudere transcriptvensters te bladeren.
- `sessions_history` verwijdert **geen** redeneringstags, `<relevant-memories>`-steigers of toolaanroep-XML uit berichttekst — het retourneert gestructureerde inhoudsblokken die dicht bij de onbewerkte transcriptvorm liggen, maar dan gemaskeerd en in grootte begrensd. `/subagents log` past de zwaardere prozareiniging toe (verwijdert redeneringstags, geheugensteigers en toolaanroep-XML), omdat het gewone chatregels weergeeft in plaats van gestructureerde blokken.
- Inspectie van het onbewerkte transcript op schijf is de terugvaloptie wanneer je het volledige transcript byte voor byte nodig hebt.

## Toolbeleid

Subagents gebruiken eerst hetzelfde profiel en dezelfde toolbeleidspijplijn als de ouder- of
doelagent. Daarna past OpenClaw de beperkingslaag voor subagents
toe.

Subagents verliezen altijd `gateway`, `agents_list`, `session_status` en
`cron`, ongeacht diepte of rol (tools op systeemniveau/interactieve tools, of
tools die de hoofdagent moet coördineren). Eind-subagents (standaardgedrag op diepte 1
en altijd op diepte 2) verliezen daarnaast `subagents`,
`sessions_list`, `sessions_history` en `sessions_spawn`. Subagents krijgen nooit
de tool `message` — deze wordt tijdens het starten uitgeschakeld, niet gefilterd door
deze weigeringslijst — en `sessions_send` blijft geweigerd, zodat subagents
alleen via de aankondigingsketen communiceren.

`sessions_history` blijft ook hier een begrensde, gereinigde terugblikweergave — het
is geen onbewerkte transcriptdump.

Wanneer `maxSpawnDepth >= 2` ontvangen orchestrator-subagents op diepte 1 daarnaast
`sessions_spawn`, `subagents`, `sessions_list` en
`sessions_history`, zodat ze hun kinderen kunnen beheren.

### Overschrijven via configuratie

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // weigeren heeft voorrang
        deny: ["gateway", "cron"],
        // als allow is ingesteld, worden alleen toegestane tools gebruikt (weigeren heeft nog steeds voorrang)
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

`tools.subagents.tools.allow` is een definitief filter dat alleen toestaat. Het kan de
reeds bepaalde toolset beperken, maar het kan een door
`tools.profile` verwijderde tool niet **opnieuw toevoegen**. `tools.profile: "coding"` bevat
bijvoorbeeld `web_search`/`web_fetch`, maar niet de tool `browser`. Om
sub-agents met het coderingsprofiel browserautomatisering te laten gebruiken, voeg je browser toe in de
profielfase:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

Gebruik `agents.entries.*.tools.alsoAllow: ["browser"]` per agent wanneer slechts één
agent browserautomatisering moet krijgen.

## Gelijktijdigheid

Sub-agents gebruiken een speciale wachtrijbaan binnen het proces:

- **Baannaam:** `subagent`
- **Gelijktijdigheid:** `agents.defaults.subagents.maxConcurrent` (standaard `8`)

## Activiteit en herstel

OpenClaw beschouwt de afwezigheid van `endedAt` niet als permanent bewijs dat een
sub-agent nog actief is. Niet-beëindigde uitvoeringen die ouder zijn dan het venster voor verouderde uitvoeringen
(2 uur, of de geconfigureerde uitvoeringstime-out plus een korte respijtperiode,
afhankelijk van welke langer is) tellen niet meer als actief/in behandeling in `/subagents list`,
statusoverzichten, blokkering van voltooiing van afstammelingen en controles van
gelijktijdigheid per sessie.

Na een herstart van de Gateway worden herstelde, verouderde, niet-beëindigde uitvoeringen verwijderd, tenzij
hun onderliggende sessie is gemarkeerd als `abortedLastRun: true`. Door een herstart afgebroken
uitvoeringen blijven geregistreerd voor de herstelprocedure voor verweesde sub-agents: verouderde
uitvoeringen worden zonder hervatting voltooid, terwijl recente onderliggende sessies
een synthetisch hervattingsbericht ontvangen voordat de afgebroken-markering wordt gewist.

Automatisch herstel na een herstart is begrensd per onderliggende sessie. Als dezelfde
onderliggende sub-agent binnen het venster voor snel opnieuw vastlopen herhaaldelijk voor verweesd herstel
wordt geaccepteerd, slaat OpenClaw een hersteltombstone op voor die
sessie en wordt deze bij latere herstarts niet meer automatisch hervat. Voer
`openclaw tasks maintenance --apply` uit om het taakrecord te reconciliëren, of
`openclaw doctor --fix` om verouderde afgebroken-herstelmarkeringen te wissen voor
sessies met een tombstone.

<Note>
Als het starten van een sub-agent mislukt met Gateway `PAIRING_REQUIRED` /
`scope-upgrade`, controleer dan de RPC-aanroeper voordat je de koppelingsstatus bewerkt.
Interne `sessions_spawn`-coördinatie wordt binnen het proces verzonden wanneer de
aanroeper al binnen de Gateway-aanvraagcontext wordt uitgevoerd, zodat er
geen WebSocket-teruglus wordt geopend en deze niet afhankelijk is van de basisomvang
van gekoppelde apparaten van de CLI. Aanroepers buiten het Gateway-proces gebruiken nog steeds de WebSocket-
terugval als `client.id: "gateway-client"` met `client.mode: "backend"`
via directe gedeelde-token-/wachtwoordauthenticatie over de teruglus. Externe aanroepers, expliciete
`deviceIdentity`, expliciete apparaattokenpaden en browser-/Node-clients
hebben nog steeds de normale apparaatgoedkeuring nodig voor uitbreidingen van de omvang.
</Note>

## Stoppen

- Het verzenden van `/stop` in de chat van de aanvrager breekt de aanvragersessie af en stopt alle actieve sub-agentuitvoeringen die daaruit zijn gestart, doorlopend naar geneste onderliggende agents.

## Beperkingen

- Aankondigingen van sub-agents gebeuren naar **beste vermogen**. Als de Gateway opnieuw wordt gestart, gaat werk dat wacht om terug te melden verloren.
- Sub-agents delen nog steeds dezelfde procesbronnen van de Gateway; beschouw `maxConcurrent` als een veiligheidsklep.
- `sessions_spawn` is altijd niet-blokkerend: het retourneert `{ status: "accepted", runId, childSessionKey }` onmiddellijk.
- De context van sub-agents injecteert alleen `AGENTS.md` en `TOOLS.md` (geen `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `HEARTBEAT.md` of `BOOTSTRAP.md`). Sube-agents die native zijn voor Codex volgen dezelfde grens: `TOOLS.md` blijft in de overgenomen Codex-threadinstructies, terwijl persona-, identiteits- en gebruikersbestanden die alleen voor de bovenliggende agent gelden, worden geïnjecteerd als samenwerkingsinstructies voor de huidige beurt, zodat onderliggende agents ze niet klonen.
- De maximale nestdiepte is 5 (bereik van `maxSpawnDepth`: 1-5). Diepte 2 wordt aanbevolen voor de meeste gebruiksscenario's.
- `maxChildrenPerAgent` begrenst actieve onderliggende agents per sessie (standaard `5`, bereik `1-20`).

## Gerelateerd

- [Sessietools en statuswijzigingen](/nl/concepts/session-tool)
- [ACP-agents](/nl/tools/acp-agents)
- [Agent verzenden](/nl/tools/agent-send)
- [Achtergrondtaken](/nl/automation/tasks)
- [Sandboxtools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools)
