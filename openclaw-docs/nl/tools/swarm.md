---
read_when:
    - Je wilt dat een Code Mode-script werk over meerdere agents verdeelt
    - Je hebt gestructureerde onderliggende resultaten, beslissingspoorten of pijplijnen nodig waarbij het eerste voltooide resultaat wordt gebruikt
    - Je schakelt limieten voor tools.swarm in of stelt ze af
    - Je wilt onderliggende collectors observeren in het sessiedashboard
sidebarTitle: Swarm
summary: Orkestreer gelijktijdige subagents vanuit Code Mode-scripts met gestructureerde resultaten, begrensde fan-out en livevoortgang
title: Zwerm
x-i18n:
    generated_at: "2026-07-27T06:37:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm is een experimentele, optionele manier om veel subagents te orkestreren vanuit een
[Code Mode](/tools/code-mode)-script. Gebruik normale JavaScript- of TypeScript-
besturingsstromen zoals `Promise.all`, `while` en `if` om werk uit te waaieren, resultaten te verzamelen
en beslissingen te nemen.

Er is geen grafiek-DSL en geen afzonderlijke workflowindeling. Het programma is de
orkestratie. Swarm voegt wachtbare collector-kinderen, gestructureerde resultaten,
begrensde gelijktijdigheid en voortgangsrapportage toe aan dat programma.

## Swarm inschakelen

Het aanbevolen pad is **Settings → Labs → Swarm** in de Control UI. De
schakelaar wordt onmiddellijk van kracht en schrijft `tools.swarm.enabled` naar je
configuratie.

Je kunt Swarm ook rechtstreeks inschakelen in `openclaw.json`:

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

De booleaanse verkorte notatie schakelt de functie in of uit, waarbij alle andere waarden
hun standaardwaarden behouden:

```json5
{
  tools: {
    swarm: true,
  },
}
```

| Veld                    | Standaard | Beschrijving                                                                                                                   |
| ----------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | Maakt spawnopties voor de collectormodus, `agents_wait` en de gast-API `agents.*` van Code Mode beschikbaar.                  |
| `maxConcurrent`         | `8`     | Maximumaantal collector-kinderen dat gelijktijdig binnen één Swarm-groep wordt uitgevoerd. Aanvullende geaccepteerde kinderen worden in FIFO-volgorde in de wachtrij geplaatst. |
| `maxChildrenPerGroup`   | `50`    | Maximumaantal actieve collector-kinderen in één groep.                                                                          |
| `maxTotalPerGroup`      | `200`   | Maximumaantal collector-kinderen dat een groep gedurende zijn levensduur mag starten. Dit is de laatste beveiliging tegen ongebreideld starten. |
| `waitTimeoutSecondsMax` | `600`   | Maximale time-out die door één aanroep van `agents_wait` wordt geaccepteerd. De standaardwaarde van de aanroep is 30 seconden. |
| `defaultAgentId`        | `""`    | Doelagent die wordt gebruikt wanneer een spawn `agentId` weglaat. Bij een lege waarde wordt de aanvragende agent gebruikt. Bestaande toelatingslijsten voor subagents zijn van toepassing. |

Numerieke waarden moeten positieve gehele getallen zijn. OpenClaw begrenst
`maxConcurrent` tot `1`–`1000`, `maxChildrenPerGroup` tot `1`–`10000`,
`maxTotalPerGroup` tot `1`–`100000` en `waitTimeoutSecondsMax` tot
`1`–`86400`.

Je kunt Swarm voor één geconfigureerde agent overschrijven met
`agents.entries.*.tools.swarm`. Het object per agent wordt over het object
`tools.swarm` op het hoogste niveau samengevoegd.

## Vereisten

De gastglobalen `agents.run`, `phase` en `log` vereisen zowel Swarm als
OpenClaw Code Mode:

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

Code Mode moet ook effectief toegang hebben tot `sessions_spawn`. Toolprofielen,
toestaan/weigeren-beleid, providerregels en sandboxbeleid kunnen die tool verwijderen.
Zie [Code Mode activeren](/tools/code-mode#activation) en
[Subagents](/nl/tools/subagents) als een script meldt dat `sessions_spawn` niet
beschikbaar is.

`defaultAgentId` en `agentId`-waarden per uitvoering moeten een geconfigureerd doel benoemen
dat is toegestaan door het `subagents.allowAgents`-beleid van de aanvrager. OpenClaw weigert
een onbekend of niet-toegestaan doel in plaats van terug te vallen op een andere agent.

## Een Swarm-script schrijven

Wanneer Swarm is ingeschakeld, stelt Code Mode deze gast-API beschikbaar:

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

Zonder `schema` wordt `agents.run()` omgezet in de uiteindelijke tekst van het kind. Met een
JSON Schema wordt de waarde gebruikt die via de tool `structured_output` van het kind
is ingediend. Een mislukt, beëindigd, verlopen of schema-ongeldig kind
wijst de promise af met een `SwarmAgentError`. Lees de exact gegenereerde
declaraties en korte orkestratiepatronen in `API.read("agents.d.ts")`
binnen Code Mode.

Gebruik `label` voor een herkenbare kindnaam in het dashboard en de zijbalk. Gebruik
`phase` in de opties om een fase te publiceren vlak voordat dat kind
begint, of roep `phase()` aan wanneer meerdere kinderen tot dezelfde fase behoren.
`log()` publiceert een korte voortgangsmelding. Voortgangsaanroepen zijn fire-and-forget;
ze vertragen het script niet als de UI niet beschikbaar is.

### Parallel uitwaaieren met gestructureerde resultaten

Dit voorbeeld start één onderzoeker per onderwerp, wacht op alle onderzoekers en
vraagt vervolgens een laatste kind om hun gestructureerde rapporten samen te voegen:

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["authentication", "storage", "recovery"];
phase("Onafhankelijke beoordeling");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`Beoordeel het pad ${topic}. Geef één bevinding met bewijs terug.`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("Synthese");
log(`${reports.length} onafhankelijke rapporten verzameld.`);

return await agents.run(
  `Breng deze rapporten met elkaar in overeenstemming en licht meningsverschillen toe:\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` is de grens voor uitwaaieren en weer samenkomen. OpenClaw start maximaal
`maxConcurrent` kinderen voor de groep en plaatst de rest in de volgorde van indiening
in de wachtrij.

Code Mode begrenst gelijktijdige aanroepen van de gastbridge afzonderlijk met
`tools.codeMode.maxPendingToolCalls` (standaard `16`, maximaal `128`). Start voor zeer
grote groepen begrensde batches onder die limiet en houd ruimte over voor
`phase()`, `log()` en wachtovergangen van kinderen. `maxConcurrent` beperkt uitgevoerde
kinderen; het verhoogt de limiet voor gastbridge-aanroepen niet.

### Een beslissingspoort herhaaldelijk controleren

Gebruik een begrensde `while`-lus wanneer elke doorgang bepaalt of nog een doorgang
nodig is:

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "Niet gecontroleerd", nextAction: "Beoordelen" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`Beslissingsdoorgang ${pass}`);
  decision = await agents.run(
    `Controleer of het releasebewijs volledig is. Vorige beslissing: ${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`Poort nog steeds gesloten na ${pass} doorgangen: ${decision.nextAction}`);
}

return decision;
```

Begrens beslissingslussen altijd. `maxTotalPerGroup` is de laatste veiligheidsgrens,
geen vervanging voor een duidelijke stopvoorwaarde.

### Het eerste kind verwerken dat klaar is

`agents.run()` retourneert een gewone promise, zodat `Promise.race` kan reageren op het
eerste Code Mode-kind. Voor harnesses die de tools op lager niveau aanroepen,
biedt `agents_wait` dezelfde grens voor de eerste voltooiing: deze retourneert zodra
ten minste één aangevraagde uitvoering is voltooid, of wanneer de begrensde time-out verloopt.
Zie [Swarm vanuit andere harnesses gebruiken](#use-swarm-from-other-harnesses) voor de
volledige afhandelingslus.

## Gedrag van collector-kinderen

Collector-kinderen zijn gewone geïsoleerde subagentsessies met een ander
voltooiingspad. Ze schrijven een duurzaam collectorresultaat waarop de ouder kan
wachten, in plaats van een antwoord terug naar de oudersessie aan te kondigen of te sturen.

De doelagent wordt in deze volgorde bepaald:

1. `agentId` bij de spawn of `agents.run()`-aanroep.
2. `tools.swarm.defaultAgentId`.
3. De aanvragende agent.

Een specifieke, lichtgewicht werkagent is nuttig wanneer Swarm-kinderen een kleiner
tooloppervlak, goedkoper model of strenger sandboxbeleid nodig hebben. OpenClaw levert
geen ingebouwde agent-id `worker`; configureer er een voordat je deze als standaard benoemt.
Versterk die worker met `tools.swarm: false` in de configuratie per agent, zodat
deze kan worden gestart maar geen swarms kan starten vanuit zijn eigen sessies op het hoogste niveau:

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

Goedkeuringen van collectors falen gesloten. Een kind opent nooit een
goedkeuringsprompt voor een operator. Een toolactie waarvoor goedkeuring nodig zou zijn,
wordt geweigerd en het kind kan die weigering in zijn resultaat melden, zodat het script
kan bepalen wat vervolgens moet gebeuren.

Voor gestructureerde uitvoer voegt OpenClaw een synthetische tool `structured_output` toe aan
het kind en valideert de payload ervan aan de hand van het opgegeven JSON Schema. Een
ongeldige of ontbrekende payload krijgt één corrigerende aansporing. Als de nieuwe poging nog steeds
niet valideert, behoudt de collectorvoltooiing de onbewerkte tekst van het kind, laat
`structured` oningesteld en neemt `schemaError` op. Het resultaat `agents_wait` op
laag niveau stelt die velden beschikbaar voor expliciete herstellogica.

### Kinderen zijn bladeren

Swarm-kinderen zijn standaard bladeren. De universele beveiliging
`agents.defaults.subagents.maxSpawnDepth` voorkomt dat een kind zijn
eigen kinderen start bij de standaarddiepte van `1`. Het gebruikelijke orkestratiepatroon is
om werk terug te geven aan de ouder, niet om meer werk vanuit een kind te starten:

```javascript
const plan = await agents.run("Plan deze taak als onafhankelijke taken.", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

Geneste subagents zijn een optionele keuze van de operator via
`agents.defaults.subagents.maxSpawnDepth` en worden afgeraden voor Swarm.
Groepslimieten, budgetten en observeerbaarheid gaan allemaal uit van platte collectorgroepen.

Elk kind heeft één toelatingseigenaar. Aankondigings- en interactieve kinderen gebruiken
`agents.defaults.subagents.maxChildrenPerAgent` (standaard `5`) en tellen
collector-kinderen niet mee. Collector-kinderen gebruiken alleen `maxChildrenPerGroup` en
`maxTotalPerGroup`; ze verbruiken het kindbudget per sessie niet. De beveiliging voor
spawndiepte blijft voor beide modi gelden.

Na toelating worden kinderen boven `maxConcurrent` in FIFO-volgorde in de wachtrij geplaatst binnen hun Swarm-
groep, genest in de globale subagentbaan. Deze gelijktijdigheidslagen plaatsen
werk in een wachtrij in plaats van het te weigeren. Een collectorspawn die een van beide groepslimieten
overschrijdt, wordt geweigerd met de relevante configuratiesleutel in de foutmelding.

## Een Swarm observeren

Open het dashboard van de oudersessie in de Control UI terwijl een Swarm actief is.
De Swarm-widget geeft elke actieve collectorgroep weer als één stip per kind, met de
status in wachtrij, actief, voltooid of mislukt. Labels verschijnen in knopinfo bij stippen, waardoor korte,
stabiele labels grotere swarms gemakkelijker leesbaar maken.

De sessiezijbalk behoudt de normale ouder/kind-boom. Vouw de ouderrij uit
om een collector-kind te bekijken of het transcript ervan te openen zonder de Swarm-
hiërarchie te verliezen.

Collectorresultaten blijven beschikbaar om op te wachten totdat hun groep is gearchiveerd. Nadat elk
lid zijn bewaartermijn heeft bereikt, archiveert OpenClaw de onderliggende processen van de groep
als batch, zodat voltooide swarms niet in de actieve sessiestructuur blijven staan.

## Swarm gebruiken vanuit andere harnassen

Je kunt Swarm gebruiken zonder OpenClaw Code Mode. De kerntools zijn
onafhankelijk van het harnas: start onderliggende collectorprocessen met
`sessions_spawn({ collect: true })` en haal hun resultaten op met begrensde `agents_wait`-
aanroepen.

Codex Code Mode stelt geschikte dynamische OpenClaw-tools automatisch beschikbaar onder
`tools.*`. Het gebruikt niet de QuickJS-gast-API van OpenClaw en vereist
`tools.codeMode` niet, maar `tools.swarm` moet nog steeds zijn ingeschakeld. `agents_wait`-
aanroepen van het Codex-harnas ondersteunen de volledige time-out van 600 seconden.

Met de momenteel ondersteunde Codex-runtime bereiken resultaten van dynamische OpenClaw-tools
Code Mode als JSON-tekst. Parseer elk resultaat voordat je velden uitleest. Codex
serialiseert dynamische toolaanroepen ook, zodat `Promise.all` niet meerdere
`sessions_spawn`-aanroepen gelijktijdig indient. Start collectors in een begrensde lus;
reeds geaccepteerde onderliggende processen kunnen blijven draaien terwijl latere starts worden ingediend.

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "Controleer het authenticatiepad.",
  "Controleer het opslagpad.",
  "Controleer het herstelpad.",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "Het starten van de collector is niet geaccepteerd.");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // Schuif dit begrensde venster achter id's die nog niet zijn gecontroleerd.
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // Verwerk elk resultaat zodra het gereed is.
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

Elke `agents_wait`-aanroep accepteert 1–1000 uitvoerings-id's. De aanroep retourneert:

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

De aanroep retourneert onmiddellijk wanneer een aangevraagd onderliggend proces al is voltooid,
wanneer ten minste één proces in behandeling wordt voltooid, wanneer er geen geldige id's in behandeling overblijven,
of wanneer de time-out verstrijkt. Voltooide records zijn idempotent, dus wanneer je de
uitvoerings-id van een reeds voltooid proces doorgeeft, wordt het resultaat opnieuw geretourneerd. Alleen de sessie
die de collector heeft gestart of de geautoriseerde bovenliggende keten daarvan kan op een collector wachten.

Dit is begrensde longpolling, geen actieve statuslus. Blijf alleen de
resterende uitvoerings-id's doorgeven totdat `pending` leeg is. De collectormodus ondersteunt native
OpenClaw-sub-agents; deze ondersteunt geen ACP-runtime, threadbinding, zichtbare
sessies of persistente sessiemodus.

## Limieten en roadmap

Swarm v1 voert eenmalige onderliggende collectorprocessen uit; de geplande `agents.session()`-API
voegt stateful workers met meerdere beurten toe. Onderliggende processen worden momenteel uitgevoerd op de
sub-agentbaan van de lokale Gateway; cloudplaatsing is gepland als een expliciete
startoptie. Opgeslagen workflowdefinities en een grafiek-DSL maken geen deel uit van de
huidige richting van Swarm.

## Gerelateerd

- [Code Mode](/tools/code-mode) voor de QuickJS-gastruntime en activeringsregels
- [Sub-agents](/nl/tools/subagents) voor beleid voor onderliggende processen, isolatie en sessiegedrag
- [Sandboxtools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) voor beperkingen per agent
- [Tooloverzicht](/nl/tools) voor toolprofielen en beleidsroutering
