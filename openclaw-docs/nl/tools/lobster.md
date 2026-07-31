---
read_when:
    - Je wilt deterministische workflows met meerdere stappen en expliciete goedkeuringen
    - Je moet een workflow hervatten zonder eerdere stappen opnieuw uit te voeren
summary: Getypeerde workflowruntime voor OpenClaw met hervatbare goedkeuringspoorten.
title: Lobster
x-i18n:
    generated_at: "2026-07-27T06:15:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85b7900f86bfedc9d73fcc91c3d0dac37b81f7413b1e68c54dd8a797b70f79fc
    source_path: tools/lobster.md
    workflow: 16
---

Lobster voert meerstaps toolpijplijnen uit als één deterministische toolaanroep, met
expliciete goedkeuringscontrolepunten en hervattingstokens. Het bevindt zich één laag boven
losgekoppeld achtergrondwerk: voor het orkestreren van flows over veel losgekoppelde taken,
zie [Task Flow](/nl/automation/taskflow) (`openclaw tasks flow`); voor het
activiteitenlogboek van taken, zie [Achtergrondtaken](/nl/automation/tasks).

## Waarom

Zonder Lobster betekent een meerstapstaak veel heen-en-weergaande toolaanroepen, waarbij het
model elke stap orkestreert. Lobster verplaatst die orkestratie naar een getypeerde
runtime:

- **Eén aanroep in plaats van vele**: één Lobster-toolaanroep retourneert een gestructureerd
  resultaat voor de volledige pijplijn.
- **Ingebouwde goedkeuringen**: neveneffecten (verzenden, plaatsen, verwijderen) stoppen de workflow
  totdat deze expliciet zijn goedgekeurd.
- **Hervatbaar**: een gestopte workflow retourneert een token; keur goed en hervat zonder
  eerdere stappen opnieuw uit te voeren.

Lobster is een kleine, beperkte DSL in plaats van een algemene scripttaal:
goedkeuren/hervatten is een duurzame, ingebouwde primitieve bewerking; pijplijnen zijn data (eenvoudig te
loggen, vergelijken, herhalen en beoordelen); de kleine grammatica beperkt "creatieve" codepaden, zodat
validatie realistisch blijft; time-outs, uitvoerlimieten, sandboxcontroles en
toegestane lijsten worden door de runtime afgedwongen, niet door elk script. Elke stap kan nog steeds
elke CLI of elk script aanroepen - genereer desgewenst `.lobster`-bestanden vanuit andere tooling als je
een rijkere auteurstaal wilt.

Zonder Lobster ziet terugkerende e-mailtriage er zo uit:

```text
Gebruiker: "Controleer mijn e-mail en stel antwoorden op"
→ openclaw roept gmail.list aan
→ LLM vat samen
→ Gebruiker: "stel antwoorden op voor #2 en #5"
→ LLM stelt concepten op
→ Gebruiker: "verzend #2"
→ openclaw roept gmail.send aan
(dagelijks herhalen, zonder geheugen van wat is getriageerd)
```

Met Lobster is dezelfde taak één aanroep die stopt voor goedkeuring en daarna wordt hervat:

```json
{ "action": "run", "pipeline": "email.triage --limit 20", "timeoutMs": 30000 }
```

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 vereisen een antwoord, 2 vereisen actie" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "2 conceptantwoorden verzenden?",
    "items": [],
    "resumeToken": "..."
  }
}
```

## Hoe het werkt

OpenClaw voert Lobster-workflows **in-process** uit met het gebundelde
`@clawdbot/lobster`-pakket als ingebedde runner. Er wordt geen extern `lobster`-
subproces gestart; de toolaanroep retourneert rechtstreeks een JSON-envelope. Als de
pijplijn stopt voor goedkeuring, bevat de envelope een hervattingstoken (of een korte
goedkeurings-ID), zodat je later kunt doorgaan.

## Inschakelen

Lobster is een **optionele** Plugin-tool en is niet standaard ingeschakeld. Deze wordt
gebundeld meegeleverd, dus er is geen afzonderlijke installatiestap vereist - sta de tool gewoon toe:

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

Of per agent:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

<Note>
`alsoAllow` voegt `lobster` toe boven op het actieve toolprofiel zonder
andere kerntools te beperken. Gebruik `tools.allow` alleen als je in plaats daarvan een beperkende
modus met een lijst van toegestane items wilt.
</Note>

De tool is volledig uitgeschakeld voor gesandboxte toolcontexten.

Als je de zelfstandige Lobster-CLI nodig hebt voor ontwikkeling of externe pijplijnen
(buiten de ingebedde Gateway-runner), installeer deze dan vanuit de
[Lobster-repository](https://github.com/openclaw/lobster) en plaats `lobster` in
`PATH`.

## Patroon: kleine CLI + JSON-pipes + goedkeuringen

Bouw kleine opdrachten die JSON gebruiken en koppel ze vervolgens aan elkaar in één Lobster-aanroep.
(Onderstaande voorbeeldopdrachtnamen kun je vervangen door je eigen namen.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Wijzigingen toepassen?'",
  "timeoutMs": 30000
}
```

Als de pijplijn om goedkeuring vraagt, hervat je deze met het token:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

Voorbeeld: invoeritems omzetten in toolaanroepen:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## LLM-stappen met uitsluitend JSON (llm-task)

Voor een **gestructureerde LLM-stap** binnen een workflow schakel je de optionele
`llm-task` Plugin-tool in en roep je deze aan vanuit Lobster:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "alsoAllow": ["llm-task"] }
      }
    ]
  }
}
```

### Belangrijke beperking: ingebedde Lobster versus `openclaw.invoke`

De gebundelde Lobster-Plugin voert workflows **in-process** uit binnen de Gateway.
In die ingebedde modus neemt `openclaw.invoke` niet automatisch een
Gateway-URL/authenticatiecontext over voor geneste OpenClaw-CLI-toolaanroepen.

Dit betekent dat dit patroon **momenteel niet betrouwbaar is in de ingebedde runner**:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Gebruik het onderstaande voorbeeld alleen als je de **zelfstandige Lobster-CLI** uitvoert in een
omgeving waarin `openclaw.invoke` al is geconfigureerd met de juiste
Gateway-/authenticatiecontext.

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Geef op basis van de ingevoerde e-mail de intentie en een concept terug.",
  "thinking": "low",
  "input": { "subject": "Hallo", "body": "Kun je helpen?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

Als je momenteel de ingebedde Lobster-Plugin gebruikt, geef dan de voorkeur aan:

- een rechtstreekse `llm-task`-toolaanroep buiten Lobster, of
- niet-`openclaw.invoke`-stappen binnen de Lobster-pijplijn totdat een ondersteunde
  ingebedde bridge is toegevoegd.

Zie [LLM-taak](/nl/tools/llm-task) voor details en configuratieopties.

## Workflowbestanden (.lobster)

Lobster kan YAML-/JSON-workflowbestanden uitvoeren met de velden `name`, `args`, `steps`, `env`,
`condition` en `approval`. Stel `pipeline` in op het bestandspad in de toolaanroep.

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

Opmerkingen:

- `stdin: $step.stdout` en `stdin: $step.json` geven de uitvoer van een eerdere stap door.
- `condition` (of `when`) kan stappen afhankelijk maken van `$step.approved`.

### Geïnjecteerde omgevingsvariabelen

Elke stapshell neemt de bovenliggende omgeving over, plus deze door Lobster geïnjecteerde
variabelen, zodat opdrachten naar opgeloste workflowargumenten kunnen verwijzen zonder
ruwe waarden in de opdrachttekenreeks op te nemen:

- `LOBSTER_ARG_<NAME>` - één per workflowargument. De naam wordt omgezet naar hoofdletters, waarbij elke
  reeks niet-alfanumerieke tekens wordt samengevoegd tot `_`, zodat argument `user-id`
  `LOBSTER_ARG_USER_ID` wordt.
- `LOBSTER_ARGS_JSON` - elk opgelost argument als één JSON-tekenreeks.

Dit is de volledige geïnjecteerde set. Er zijn **geen** uitvoervariabelen per stap,
zoals `LOBSTER_STEP_<id>_STDOUT` of `LOBSTER_STEP_<id>_JSON_<field>`; shells
behandelen die namen als niet ingesteld, waardoor standaardwaarden van parameterexpansie de fout kunnen verbergen.
Lees de uitvoer van een eerdere stap in plaats daarvan via stapverwijzingen - `$step.stdout`,
`$step.json` of `$step.json.<field>` - in een waarde voor `stdin:`, `env:` of `condition:`.
(`LOBSTER_STATE_DIR` is een afzonderlijke runtime-instelling voor de statusmap,
geen argument per uitvoering.)

## Toolparameters

### `run`

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

Een workflowbestand uitvoeren met argumenten:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

| Veld             | Standaard    | Opmerkingen                                                                                                  |
| ---------------- | ------------ | ------------------------------------------------------------------------------------------------------------ |
| `pipeline`       | vereist      | Inline pijplijntekenreeks, of een pad dat eindigt op `.lobster`/`.yaml`/`.yml`/`.json` voor een workflowbestand. |
| `cwd`            | Gateway-cwd | Relatieve werkmap; moet binnen de Gateway-werkmap worden opgelost (absolute paden worden geweigerd).          |
| `timeoutMs`      | `20000`      | Breekt de uitvoering af als dit wordt overschreden.                                                          |
| `maxStdoutBytes` | `512000`     | Breekt de uitvoering af als vastgelegde stdout of stderr deze grootte overschrijdt.                           |
| `argsJson`       | -            | JSON-tekenreeks met argumenten voor een workflowbestand (genegeerd voor inline pijplijnen).                   |

### `resume`

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

`resume` accepteert `token` (het volledige hervattingstoken uit `requiresApproval`)
of `approvalId` (de korte ID uit hetzelfde object) - gebruik wat de gestopte
uitvoering heeft geretourneerd. `approve` is vereist.

### Beheerde Task Flow-modus

Door `flowControllerId` en `flowGoal` door te geven aan `run` (of `flowId` en
`flowExpectedRevision` aan `resume`) wordt de aanroep via de beheerde
[Task Flow](/nl/automation/taskflow)-API van de Plugin-runtime uitgevoerd, in plaats van
een kale envelope te retourneren: OpenClaw maakt of hervat een duurzaam flowrecord, past de
Lobster-envelope erop toe (`waiting` bij goedkeuring, `succeeded`/`failed` bij
voltooiing) en retourneert `{ ok, envelope, flow, mutation }`. Deze modus vereist
een gekoppelde Task Flow-runtime en is bedoeld voor Plugin-/controllercode die
duurzame flowstatus nodig heeft na Gateway-herstarts, niet voor normaal ad-hocgebruik door agents.

## Uitvoerenvelope

Lobster retourneert een JSON-envelope met een van drie statussen:

- `ok` - succesvol voltooid
- `needs_approval` - gepauzeerd; `requiresApproval` bevat een `resumeToken` en een
  korte `approvalId`, waarmee de uitvoering beide kan worden hervat
- `cancelled` - expliciet geweigerd of geannuleerd

De tool maakt de envelope beschikbaar in zowel `content` (opgemaakte JSON) als `details`
(ruw object).

## Goedkeuringen

Als `requiresApproval` aanwezig is, controleer je de prompt en beslis je:

- `approve: true` - hervatten en doorgaan met neveneffecten
- `approve: false` - de workflow annuleren en afronden

Gebruik `approve --preview-from-stdin --limit N` om een JSON-voorbeeld aan
goedkeuringsverzoeken toe te voegen zonder aangepaste jq-/heredoc-constructies. De hervattingsstatus wordt opgeslagen als
kleine JSON-bestanden in de Lobster-statusmap (standaard `~/.lobster/state`,
te overschrijven met `LOBSTER_STATE_DIR`); het token zelf codeert alleen een
verwijzing naar die status, niet de volledige pijplijnstatus.

## OpenProse

OpenProse werkt goed samen met Lobster: gebruik `/prose` om voorbereiding door meerdere agents
te orkestreren en voer vervolgens een Lobster-pijplijn uit voor deterministische goedkeuringen. Als een Prose-
programma Lobster nodig heeft, sta je de tool `lobster` toe voor subagents via
`tools.subagents.tools`. Zie [OpenProse](/nl/prose).

## Veiligheid

- **Alleen lokaal binnen hetzelfde proces** - workflows worden uitgevoerd binnen het Gateway-proces; geen
  netwerkaanroepen vanuit de Plugin zelf.
- **Geen geheimen** - Lobster beheert OAuth niet; het roept OpenClaw-tools aan die
  dat wel doen.
- **Sandboxbewust** - uitgeschakeld wanneer de toolcontext in een sandbox wordt uitgevoerd.
- **Versterkt** - time-outs en uitvoerlimieten worden afgedwongen door de ingebouwde runner.

## Probleemoplossing

| Fout                                                          | Oorzaak / oplossing                                                               |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `lobster runtime timed out`                                   | De pijplijn overschreed `timeoutMs`. Verhoog deze waarde of splits de pijplijn. |
| `lobster stdout exceeded maxStdoutBytes` (of `stderr`)        | De vastgelegde uitvoer overschreed de limiet. Verhoog `maxStdoutBytes` of verminder de uitvoer. |
| `run --args-json must be valid JSON`                          | `argsJson` (uitvoeringen vanuit workflowbestanden) kon niet worden geparseerd. Corrigeer de JSON-tekenreeks. |
| `lobster runtime failed` (of een ander `runtime_error`-bericht) | De ingebouwde runtime retourneerde een foutenvelop. Controleer de Gateway-logboeken voor details. |

## Meer informatie

- [Plugins](/nl/tools/plugin)
- [Tools voor Plugins maken](/nl/plugins/building-plugins#registering-agent-tools)

## Praktijkvoorbeeld: communityworkflows

Een openbaar voorbeeld: een CLI voor een 'tweede brein' met Lobster-pijplijnen die drie
Markdown-kluizen beheren (persoonlijk, partner, gedeeld). De CLI produceert JSON voor statistieken,
inboxoverzichten en scans op verouderde inhoud; Lobster koppelt die opdrachten tot workflows
zoals `weekly-review`, `inbox-triage`, `memory-consolidation` en
`shared-task-sync`, elk met goedkeuringspoorten. AI neemt waar mogelijk beslissingen
(categorisering) en valt anders terug op deterministische regels.

- Discussie: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- Repo: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## Gerelateerd

- [Automatisering](/nl/automation) - alle automatiseringsmechanismen
- [Overzicht van tools](/nl/tools) - alle beschikbare agenttools
