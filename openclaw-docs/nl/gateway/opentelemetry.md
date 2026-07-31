---
read_when:
    - Je wilt metrische gegevens over OpenClaw-modelgebruik, berichtenstromen of sessies naar een OpenTelemetry-collector verzenden
    - Je koppelt traces, metrics of logs aan Grafana, Datadog, Honeycomb, New Relic, Tempo of een andere OTLP-backend
    - Je hebt de exacte namen van metriekgegevens, spans of de structuur van attributen nodig om dashboards of waarschuwingen te maken
summary: Exporteer OpenClaw-diagnostiek naar OpenTelemetry-collectors of stdout-JSONL via de diagnostics-otel-plugin
title: OpenTelemetry-export
x-i18n:
    generated_at: "2026-07-27T05:05:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw exporteert diagnostische gegevens via de officiële `diagnostics-otel`-Plugin
met **OTLP/HTTP (protobuf)**. Logboeken kunnen ook als stdout-JSONL worden geschreven voor
logpijplijnen van containers en sandboxes. Elke collector of backend die
OTLP/HTTP accepteert, werkt zonder codewijzigingen. Zie voor lokale bestandslogboeken
[Logboekregistratie](/nl/logging).

- **Diagnostische gebeurtenissen** zijn gestructureerde records binnen het proces die worden uitgezonden door de
  Gateway en gebundelde plugins voor modeluitvoeringen, berichtenstromen, sessies, wachtrijen
  en exec.
- **`diagnostics-otel`** abonneert zich op deze gebeurtenissen en exporteert ze als
  OpenTelemetry-**metrische gegevens**, **traces** en **logboeken** via OTLP/HTTP, en kan
  logboekrecords spiegelen naar stdout-JSONL.
- **Provideraanroepen** ontvangen een W3C `traceparent`-header uit de
  vertrouwde spancontext van OpenClaw voor modelaanroepen wanneer het providertransport aangepaste
  headers accepteert. Door plugins uitgezonden tracecontext wordt niet doorgegeven.
- Exporters worden alleen gekoppeld wanneer zowel het diagnostische oppervlak als de Plugin zijn
  ingeschakeld, zodat de kosten binnen het proces standaard vrijwel nul blijven.

## Snel aan de slag

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
```

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

Of schakel de Plugin in via de CLI: `openclaw plugins enable diagnostics-otel`.

<Note>
`protocol` ondersteunt alleen `http/protobuf`. Omdat `traces` en `metrics` standaard zijn ingeschakeld, breekt elke andere waarde (inclusief `grpc`) het volledige diagnostics-otel-abonnement af met een `unsupported protocol`-waarschuwing. Hierdoor stopt ook de export van stdout-logboeken. Stel `traces: false` en `metrics: false` expliciet in als je alleen `logsExporter: "stdout"` met een niet-OTLP-protocolwaarde wilt.
</Note>

## Geëxporteerde signalen

| Signaal      | Wat het bevat                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Metrische gegevens** | Tellers/histogrammen voor tokengebruik, kosten, uitvoeringsduur, failover, gebruik van Skills, berichtenstroom, Talk-gebeurtenissen, wachtrijbanen, sessiestatus/-herstel, tooluitvoering, exec, geheugen, beschikbaarheid en gezondheid van exporters. |
| **Traces**  | Spans voor modelgebruik, modelaanroepen, de levenscyclus van de harness, gebruik van Skills, tooluitvoering, exec, verwerking van webhooks/berichten, contextopbouw en toollussen.                                                      |
| **Logboeken**    | Gestructureerde `logging.file`-records die via OTLP of stdout-JSONL worden geëxporteerd wanneer `diagnostics.otel.logs` is ingeschakeld; logboekinhoud wordt achtergehouden tenzij het vastleggen van inhoud expliciet is ingeschakeld.                          |

Schakel `traces`, `metrics` en `logs` onafhankelijk in of uit. Traces en metrische gegevens
zijn standaard ingeschakeld wanneer `diagnostics.otel.enabled` waar is; logboeken zijn standaard uitgeschakeld
en worden alleen geëxporteerd wanneer `diagnostics.otel.logs` expliciet `true` is. Logboekexport
gebruikt standaard OTLP; stel `diagnostics.otel.logsExporter` in op `stdout` voor JSONL op
stdout, of op `both` voor beide.

## Configuratiereferentie

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc schakelt OTLP-export uit
      serviceName: "openclaw-gateway", // indien niet ingesteld, wordt teruggevallen op OTEL_SERVICE_NAME en vervolgens "openclaw"
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | beide
      sampleRate: 0.2, // sampler voor root-spans, 0.0..1.0
      flushIntervalMs: 60000, // exportinterval voor metrische gegevens (min. 1000ms)
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },
  },
}
```

### Omgevingsvariabelen

| Variabele                                                                                                          | Doel                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | Terugvalwaarde voor `diagnostics.otel.endpoint` wanneer de configuratiesleutel niet is ingesteld.                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | Signaalspecifieke terugval-eindpunten die worden gebruikt wanneer de overeenkomstige configuratiesleutel `diagnostics.otel.*Endpoint` niet is ingesteld. Signaalspecifieke configuratie heeft voorrang op signaalspecifieke omgevingsvariabelen, die weer voorrang hebben op het gedeelde eindpunt.                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | Terugvalwaarde voor `diagnostics.otel.serviceName` wanneer de configuratiesleutel niet is ingesteld. De standaardservicenaam is `openclaw`.                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | Terugvalwaarde voor het wire-protocol wanneer `diagnostics.otel.protocol` niet is ingesteld. Alleen `http/protobuf` schakelt export in.                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | Stel in op `gen_ai_latest_experimental` om de nieuwste vorm van de GenAI-inferentiespan uit te zenden: `{gen_ai.operation.name} {gen_ai.request.model}`-spannamen, `CLIENT`-spansoort en `gen_ai.provider.name` in plaats van de verouderde `gen_ai.system`. GenAI-metrische gegevens gebruiken hoe dan ook altijd begrensde attributen met een lage cardinaliteit. |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | Stel in op `1` wanneer een andere preload of een ander hostproces de globale OpenTelemetry-SDK al heeft geregistreerd. De Plugin slaat dan zijn eigen NodeSDK-levenscyclus over, maar koppelt nog steeds diagnostische listeners en respecteert `traces`/`metrics`/`logs`.                                                                                    |

## Privacy en het vastleggen van inhoud

Onbewerkte model-/toolinhoud wordt standaard **niet** geëxporteerd. Spans bevatten begrensde
identificatoren (kanaal, provider, model, foutcategorie, uitsluitend gehashte aanvraag-id's,
toolbron, tooleigenaar, naam/bron van de Skill) en bevatten nooit prompttekst,
antwoordtekst, toolinvoer, tooluitvoer, bestandspaden van Skills of sessiesleutels.
Waarden die eruitzien als sessiesleutels van agents met een bepaald bereik (bijvoorbeeld beginnend met
`agent:`) worden bij attributen met een lage cardinaliteit vervangen door `unknown`. OTLP-logboekrecords
behouden standaard ernstniveau, logger, codelocatie, vertrouwde tracecontext en
gezuiverde attributen; de onbewerkte inhoud van het logboekbericht wordt alleen geëxporteerd
wanneer `diagnostics.otel.captureContent` de Booleaanse waarde `true` heeft. Gedetailleerde
`captureContent.*`-subsleutels schakelen logboekinhoud nooit in. Talk-metrische gegevens exporteren alleen
begrensde metagegevens van gebeurtenissen (modus, transport, provider, gebeurtenistype), geen
transcripten, audiopayloads, sessie-id's, beurt-id's, oproep-id's, ruimte-id's of
overdrachtstokens.

Uitgaande modelaanvragen kunnen een W3C `traceparent`-header bevatten die uitsluitend is gegenereerd
op basis van diagnostische tracecontext die eigendom is van OpenClaw voor de actieve modelaanroep.
Bestaande door de aanroeper opgegeven `traceparent`-headers worden vervangen, zodat plugins of
aangepaste provideropties trace-afstamming tussen services niet kunnen vervalsen.

Stel `diagnostics.otel.captureContent.*` alleen in op `true` wanneer je collector
en bewaarbeleid zijn goedgekeurd voor prompt-, antwoord-, tool- of
systeemprompttekst. Elke subsleutel is onafhankelijk:

- `inputMessages` - inhoud van gebruikersprompts.
- `outputMessages` - inhoud van modelantwoorden.
- `toolInputs` - payloads met toolargumenten.
- `toolOutputs` - payloads met toolresultaten.
- `systemPrompt` - samengestelde systeem-/ontwikkelaarsprompt.
- `toolDefinitions` - namen, beschrijvingen en schema's van modeltools.

Wanneer een subsleutel is ingeschakeld, krijgen model- en toolspans alleen voor die klasse begrensde, geredigeerde
`openclaw.content.*`-attributen.

<Note>
De Booleaanse waarde `captureContent: true` schakelt `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `toolDefinitions` en OTLP-logboekinhoud samen in, maar **niet** `systemPrompt`. Stel `captureContent.systemPrompt: true` expliciet in als je ook de samengestelde systeemprompt nodig hebt.
</Note>

`toolInputs`/`toolOutputs`-inhoud wordt vastgelegd voor tooluitvoeringen van de ingebouwde
agentruntime (`openclaw.content.tool_input` en
`gen_ai.tool.call.arguments` bij voltooide/foutspans;
`openclaw.content.tool_output` en `gen_ai.tool.call.result` bij voltooide
spans). De `openclaw.content.*`-namen blijven de stabiele OpenClaw-attribuutnamen;
de `gen_ai.tool.call.*`-kopieën weerspiegelen ze voor semconv-native viewers.
Toolaanroepen van externe harnesses (Codex, Claude CLI) zenden
`tool.execution.*`-spans uit zonder inhoudspayloads. Vastgelegde inhoud wordt via een
vertrouwd kanaal uitsluitend voor listeners verzonden en wordt nooit op de openbare bus voor diagnostische gebeurtenissen
geplaatst.

## Sampling en doorschrijven

- **Traces:** `diagnostics.otel.sampleRate` stelt alleen op de rootspan een `TraceIdRatioBasedSampler`
  in (`0.0` verwijdert alles, `1.0` behoudt alles). Als dit niet is ingesteld, wordt de
  standaardwaarde van de OpenTelemetry SDK gebruikt (altijd ingeschakeld).
- **Metrieken:** `diagnostics.otel.flushIntervalMs` (begrensd op een minimum van
  `1000`); als dit niet is ingesteld, wordt de standaardwaarde voor periodieke export van de SDK gebruikt.
- **Logboeken:** OTLP-logboeken respecteren `logging.level` (logniveau van bestanden) en gebruiken het
  redactiepad voor diagnostische logrecords, niet de consoleopmaak. Installaties met een hoog volume
  kunnen beter sampling/filtering via de OTLP-collector gebruiken dan lokale
  sampling. Stel `diagnostics.otel.logsExporter: "stdout"` in wanneer je platform
  stdout/stderr al naar een logverwerker verzendt en je geen collector voor OTLP-logboeken
  hebt. Stdout-records bestaan uit één JSON-object per regel met `ts`, `signal`,
  `service.name`, ernst, inhoud, geredigeerde attributen en vertrouwde tracevelden
  wanneer die beschikbaar zijn.
- **Correlatie van bestandslogboeken:** JSONL-bestandslogboeken bevatten `traceId`,
  `spanId`, `parentSpanId` en `traceFlags` op het hoogste niveau wanneer de logaanroep een geldige
  diagnostische tracecontext bevat, zodat logverwerkers lokale logregels aan
  geëxporteerde spans kunnen koppelen.
- **Correlatie van aanvragen:** HTTP-aanvragen en WebSocket-frames van de Gateway maken
  een intern tracebereik voor aanvragen. Logboeken en diagnostische gebeurtenissen binnen dat
  bereik nemen standaard de trace van de aanvraag over, terwijl spans voor agentruns en modelaanroepen
  als onderliggende spans worden gemaakt, zodat `traceparent`-headers van de provider binnen
  dezelfde trace blijven.
- **Correlatie van modelaanroepen:** `openclaw.model.call`-spans bevatten standaard veilige groottes van
  promptcomponenten en tokenattributen per aanroep wanneer het providerresultaat
  gebruiksgegevens beschikbaar stelt. `openclaw.model.usage` blijft de span op runniveau
  voor de geaggregeerde kosten-, context- en kanaaldashboards en
  blijft binnen dezelfde diagnostische trace wanneer de emitterende runtime een vertrouwde
  tracecontext heeft.

### Observatie-eenheden voor modelaanroepen

Elke `openclaw.model.call`-span geeft via
`openclaw.model_call.observation_unit` aan wat de levenscyclus ervan meet:

- `request` - één waarneembare model-/provideraanvraag. Ingebedde systeemeigen modelaanroepen
  gebruiken deze eenheid en exporters behandelen een ontbrekende waarde als `request` voor
  compatibiliteit met oudere of externe emitters.
- `turn` - één ondoorzichtige beurt van de agent-CLI die verborgen modelaanvragen,
  nieuwe pogingen, toolwerk of achtergrondwerk kan bevatten. Aanroepen van de Claude Code CLI en Codex-appserver
  gebruiken deze eenheid.

Beide eenheden blijven spans voor modelaanroepen, zodat tracebackends modelinvoer,
-uitvoer, gebruik en hiërarchie kunnen weergeven. Aanvraagspans gebruiken de van de API afgeleide GenAI-bewerking
(`chat`, `generate_content` of `text_completion`), terwijl beurtspans
`gen_ai.operation.name = invoke_agent` gebruiken. Beide dragen bij aan
`gen_ai.client.operation.duration`, waarbij de bewerkingsnaam de latentie van directe
aanvragen gescheiden houdt van de latentie van volledige beurten. De OTEL-metrieken voor modelaanroepen van OpenClaw
bevatten ook `openclaw.model_call.observation_unit`; de Prometheus-metrieken
voor modelaanroepen stellen het equivalente label `observation_unit` beschikbaar.

### Nauwkeurigheid van modelaanroepen in de Claude Code CLI

Beurten van de Claude Code CLI emitten één synthetische `openclaw.model.call`-span
op beurtniveau. Dit zijn geen HTTP-aanvraagspans van Anthropic. Ze gebruiken `openclaw.api =
claude-code`, `openclaw.model_call.observation_unit = turn` en duiden
de bewerking aan als `gen_ai.operation.name = invoke_agent`. Ze identificeren
de CLI-grens van OpenClaw via
`openclaw.transport`:

- `stdio` - een eenmalig lokaal Claude Code-proces.
- `stdio-live` - één beurt in een beheerde, persistente Claude-stdio-sessie.
- `paired-node-cli` - een eenmalige uitvoering van Claude Code die aan een gekoppelde
  Node is gedelegeerd.

Diagnostiek voor de Claude CLI wordt alleen geïnstantieerd wanneer de diagnostische
dispatcher van het proces is ingeschakeld en een interne of vertrouwde gebeurtenislistener is gekoppeld.
Als er geen observability-plugin of andere listener actief is, slaan Claude CLI-beurten
de synthetische tracehiërarchie, inhoudsbuffers en diagnostische boekhouding van
streambytes over. Wanneer inhoudsregistratie is ingeschakeld, zijn de velden voor de prompt en systeemprompt
elk beperkt tot 128 KiB; assistentuitvoer is beperkt tot 128 KiB verdeeld over maximaal
200 enveloppen, waarbij 16 KiB en één item zijn gereserveerd voor een laatste zichtbare
fallbackreactie. Een markering registreert afkapping wanneer de limiet wordt bereikt.

OpenClaw geeft Claude CLI-beurten dezelfde eigendomshiërarchie als andere
agentruntimes: `openclaw.harness.run` (`openclaw.harness.id = claude-cli`)
bevat `openclaw.run`, die de Claude-`openclaw.model.call`-span bevat.
De harness- en runspans zijn synthetische OpenClaw-grenzen voor beurten, geen
interne fasen van Claude Code. Eenmalige en beheerde stdio-beurten gebruiken dezelfde
hiërarchie; een echte nieuwe poging met een nieuwe sessie maakt nog een onderliggende modelaanroep binnen
dezelfde OpenClaw-run.

De span begint wanneer OpenClaw de voorbereide CLI-beurt accepteert en eindigt pas nadat
die beurt slaagt of mislukt. Voor beheerde sessies beëindigt een tussentijds succesvol resultaat
de span niet zolang Claude achtergrondagents of
workflows meldt die het resultaat vasthouden; het uiteindelijke resultaat na het leegmaken doet dat wel. Afbreken, time-out, procesfout,
uitvoer-/parseerfout en andere fouten tijdens een beurt beëindigen dezelfde span met een fout.

Claude Code rapporteert gebruik per assistentbericht en kan ook cumulatief
gebruik rapporteren in het eindresultaat. De antwoordboekhouding van OpenClaw blijft het
laatste assistentbericht gebruiken, zodat de bestaande kostensemantiek niet verandert; de
span voor modelaanroepen op beurtniveau gebruikt cumulatief eindgebruik wanneer beschikbaar,
inclusief tokens voor cachelezing en cacheaanmaak.

Voor deze CLI-spans beschrijven de byte- en timingvelden de waarneembare
CLI-grens van OpenClaw:

- `openclaw.model_call.request_bytes` is de UTF-8-grootte van de promptwaarde
  die via eenmalige stdin/argv of de beheerde stdio-JSONL-gebruikersenvelop wordt verzonden. Dit
  is niet de grootte van de verborgen modelaanvraag van Claude Code.
- `openclaw.model_call.response_bytes` is de UTF-8-grootte van de stdout van de Claude CLI
  die tijdens de beurt wordt waargenomen. Dit is niet de grootte van de HTTP-respons van Anthropic.
- `openclaw.model_call.time_to_first_byte_ms` is de tijd tot de eerste waarneembare
  stdout- of stderr-uitvoer van de Claude CLI. Dit is niet de TTFB van het netwerk.

Als de bijbehorende gedetailleerde `captureContent`-velden zijn ingeschakeld, exporteert de span
de effectieve prompt die OpenClaw naar Claude Code verzendt, de door OpenClaw toegevoegde systeemprompt
en zichtbare assistenttekst/redenering/identiteit van toolaanroepen via
`gen_ai.input.messages`, `gen_ai.output.messages` en
`gen_ai.system_instructions`. Toolargumenten, ondoorzichtige denksignaturen en
toolresultaten worden uit de Claude-assistentenvelop weggelaten. OpenClaw beweert geen
toegang te hebben tot de privé-systeemprompt van Claude Code, verborgen hervatte of
gecompacteerde aanvraagpayload, systeemeigen interne toolschema's, onbewerkte HTTP-aanvraag van Anthropic,
interne nieuwe pogingen, upstream-aanvraag-id of werkelijke netwerk-TTFB. Omdat
Claude Code zijn effectieve systeemeigen tooldefinities niet nauwkeurig beschikbaar stelt,
vullen deze spans `gen_ai.tool.definitions` niet in.

Externe toolspans van de Claude-harness blijven uitsluitend metadata bevatten, zelfs wanneer het registreren van
toolinhoud is ingeschakeld. Net als bij elke modelspan gebruikt vastgelegde Claude CLI-inhoud
het pad dat uitsluitend voor vertrouwde listeners beschikbaar is en de bestaande redactie- en groottebegrenzingen
van de exporter; inhoud blijft standaard uitgeschakeld.

## Geëxporteerde metrieken

### Modelgebruik

- `openclaw.tokens` (teller, attrs: `openclaw.token`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.agent`)
- `openclaw.cost.usd` (teller, attrs: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.run.duration_ms` (histogram, attrs: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.context.tokens` (histogram, attrs: `openclaw.context`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `gen_ai.client.token.usage` (histogram, GenAI-metriek volgens semantische conventies, attrs: `gen_ai.token.type` = `input`/`output`, `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`)
- `gen_ai.client.operation.duration` (histogram, seconden, GenAI-metriek volgens semantische conventies voor modelaanvragen en synthetische agentbeurten; attrs: `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`, optioneel `error.type`; observaties van beurten gebruiken `gen_ai.operation.name = invoke_agent`)
- `openclaw.model_call.duration_ms` (histogram, attrs: `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit`, plus `openclaw.errorCategory` en `openclaw.failureKind` bij geclassificeerde fouten)
- `openclaw.model_call.request_bytes` (histogram, UTF-8-bytegrootte van de uiteindelijke payload van de modelaanvraag; voor de Claude Code CLI de hierboven beschreven waarneembare promptinvoer/-envelop; geen onbewerkte payloadinhoud)
- `openclaw.model_call.response_bytes` (histogram, UTF-8-bytegrootte van gestreamde payloadfragmenten van de respons; hoogfrequente delta's voor tekst, gedachten en toolaanroepen tellen alleen incrementele `delta`-bytes; voor de Claude Code CLI waargenomen stdout-bytes; geen onbewerkte responsinhoud)
- `openclaw.model_call.time_to_first_byte_ms` (histogram, verstreken tijd vóór de eerste gestreamde responsgebeurtenis; voor de Claude Code CLI de eerste waarneembare CLI-uitvoer in plaats van netwerk-TTFB)
- `openclaw.model.failover` (teller, attrs: `openclaw.provider`, `openclaw.model`, `openclaw.failover.to_provider`, `openclaw.failover.to_model`, `openclaw.failover.reason`, `openclaw.failover.suspended`, `openclaw.lane`)
- `openclaw.skill.used` (teller, attrs: `openclaw.skill.name`, `openclaw.skill.source`, `openclaw.skill.activation`, optioneel `openclaw.agent`, optioneel `openclaw.toolName`)

### Berichtenstroom

- `openclaw.webhook.received` (teller, attrs: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.error` (teller, attrs: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.duration_ms` (histogram, attrs: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.message.queued` (teller, attrs: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.received` (teller, attrs: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.started` (teller, attrs: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.completed` (teller, attrs: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.dispatch.duration_ms` (histogram, attrs: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.processed` (teller, attrs: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.duration_ms` (histogram, attrs: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.delivery.started` (teller, attrs: `openclaw.channel`, `openclaw.delivery.kind`)
- `openclaw.message.delivery.duration_ms` (histogram, attrs: `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`)

### Talk

- `openclaw.talk.event` (teller, attrs: `openclaw.talk.event_type`, `openclaw.talk.mode`, `openclaw.talk.transport`, `openclaw.talk.brain`, `openclaw.talk.provider`)
- `openclaw.talk.event.duration_ms` (histogram, attrs: hetzelfde als `openclaw.talk.event`; geëmit wanneer een Talk-gebeurtenis een duur rapporteert)
- `openclaw.talk.audio.bytes` (histogram, attrs: hetzelfde als `openclaw.talk.event`; geëmit voor Talk-audioframegebeurtenissen die een bytelengte rapporteren)

### Wachtrijen en sessies

- `openclaw.queue.lane.enqueue` (teller, attributen: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` (teller, attributen: `openclaw.lane`)
- `openclaw.queue.depth` (histogram, attributen: `openclaw.lane` of `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` (histogram, attributen: `openclaw.lane`)
- `openclaw.session.state` (teller, attributen: `openclaw.state`, `openclaw.reason`)
- `openclaw.session.stuck` (teller, attributen: `openclaw.state`; gegenereerd voor herstelbare verouderde sessieboekhouding)
- `openclaw.session.stuck_age_ms` (histogram, attributen: `openclaw.state`; gegenereerd voor herstelbare verouderde sessieboekhouding)
- `openclaw.session.turn.created` (teller, attributen: `openclaw.agent`, `openclaw.channel`, `openclaw.trigger`)
- `openclaw.session.recovery.requested` (teller, attributen: `openclaw.state`, `openclaw.action`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.completed` (teller, attributen: `openclaw.state`, `openclaw.action`, `openclaw.status`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.age_ms` (histogram, attributen: dezelfde als de bijbehorende herstelteller)
- `openclaw.run.attempt` (teller, attributen: `openclaw.attempt`)

### Telemetrie voor sessieactiviteit

Een `processing`-sessie nadert de ingebouwde activiteitsdrempel niet zolang OpenClaw voortgang waarneemt van antwoorden, tools, status, blokkeringen of de ACP-runtime. Typindicaties om de verbinding actief te houden gelden niet als voortgang, zodat een stil model of stille harness nog steeds kan worden gedetecteerd.

OpenClaw classificeert sessies op basis van het werk dat nog kan worden waargenomen:

- `session.long_running`: actief ingebed werk, modelaanroepen of toolaanroepen
  boeken nog steeds voortgang. Stille modelaanroepen met een eigenaar worden vóór de ingebouwde afbreekdrempel ook als langlopend gerapporteerd, zodat trage of niet-streamende modelproviders niet op vastgelopen Gateway-sessies lijken zolang afbreken waarneembaar is.
- `session.stalled`: er bestaat actief werk, maar de actieve uitvoering heeft
  onlangs geen voortgang gemeld. Modelaanroepen met een eigenaar schakelen op of na de ingebouwde afbreekdrempel over van `session.long_running` naar
  `session.stalled`; verouderde model-/toolactiviteit zonder eigenaar wordt
  niet als onschadelijk langlopend werk beschouwd.
  Vastgelopen ingebedde uitvoeringen worden aanvankelijk alleen geobserveerd en gaan vervolgens
  na de afbreekdrempel zonder voortgang over op afbreken en leegmaken, zodat uitvoeringen in de wachtrij achter de lane kunnen worden hervat.
- `session.stuck`: verouderde sessieboekhouding zonder actief werk, of een inactieve
  sessie in de wachtrij met verouderde model-/toolactiviteit zonder eigenaar. Hierdoor wordt de
  betreffende sessielane onmiddellijk vrijgegeven nadat de herstelcontroles zijn geslaagd.

Herstel genereert gestructureerde `session.recovery.requested`- en
`session.recovery.completed`-gebeurtenissen. De diagnostische sessiestatus wordt alleen als inactief gemarkeerd
na een muterend herstelresultaat (`aborted` of `released`) en alleen als
dezelfde verwerkingsgeneratie nog actueel is.

Alleen `session.stuck` genereert de teller `openclaw.session.stuck`, het
histogram `openclaw.session.stuck_age_ms` en de span `openclaw.session.stuck`.
Herhaalde `session.stuck`-diagnoses passen exponentiële vertraging toe zolang de sessie
ongewijzigd blijft. Dashboards moeten daarom waarschuwen bij aanhoudende stijgingen en niet bij
elke Heartbeat-tik. Zie voor de configuratieoptie en standaardwaarden de
[configuratiereferentie](/nl/gateway/configuration-reference#diagnostics).

Activiteitswaarschuwingen genereren ook:

- `openclaw.liveness.warning` (teller, attributen: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_p99_ms` (histogram, attributen: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_max_ms` (histogram, attributen: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_utilization` (histogram, attributen: `openclaw.liveness.reason`)
- `openclaw.liveness.cpu_core_ratio` (histogram, attributen: `openclaw.liveness.reason`)

### Levenscyclus van de harness

- `openclaw.harness.duration_ms` (histogram, attributen: `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, `openclaw.harness.phase` bij fouten)

### Tooluitvoering en lusdetectie

- `openclaw.tool.execution.duration_ms` (histogram, attributen: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, plus `openclaw.errorCategory` bij fouten)
- `openclaw.tool.execution.blocked` (teller, attributen: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, `openclaw.deniedReason`)
- `openclaw.tool.loop` (teller, attributen: `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, optioneel `openclaw.loop.paired_tool`; gegenereerd wanneer een repetitieve lus van toolaanroepen wordt gedetecteerd)

### Exec

- `openclaw.exec.duration_ms` (histogram, attributen: `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`)

### Interne diagnostiek (geheugen, payloads, gezondheid van de exporter)

- `openclaw.payload.large` (teller, attributen: `openclaw.payload.surface`, `openclaw.payload.action`, `openclaw.channel`, `openclaw.plugin`, `openclaw.reason`)
- `openclaw.payload.large_bytes` (histogram, attributen: dezelfde als `openclaw.payload.large`)
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes` (histogrammen, geen attributen; metingen van procesgeheugen)
- `openclaw.memory.pressure` (teller, attributen: `openclaw.memory.level`, `openclaw.memory.reason`)
- `openclaw.diagnostic.async_queue.dropped` (teller, attributen: `openclaw.diagnostic.async_queue.drop_class`; verwijderingen door tegendruk in de interne diagnostiekwachtrij)
- `openclaw.telemetry.exporter.events` (teller, attributen: `openclaw.exporter`, `openclaw.signal`, `openclaw.status`, optioneel `openclaw.reason`, optioneel `openclaw.errorCategory`; zelftelemetrie voor de levenscyclus en storingen van de exporter)

## Geëxporteerde spans

- `openclaw.model.usage`
  - `openclaw.channel`, `openclaw.provider`, `openclaw.model`
  - `openclaw.tokens.*` (invoer/uitvoer/cache_read/cache_write/totaal)
  - `gen_ai.system` standaard, of `gen_ai.provider.name` wanneer voor de nieuwste semantische GenAI-conventies is gekozen
  - `gen_ai.request.model`, `gen_ai.operation.name`, `gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.errorCategory`
- `openclaw.model.call`
  - `gen_ai.system` standaard, of `gen_ai.provider.name` wanneer voor de nieuwste semantische GenAI-conventies is gekozen
  - `gen_ai.request.model`, `gen_ai.operation.name`, `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit` (`request` of `turn`)
  - `openclaw.errorCategory`, `error.type` en optioneel `openclaw.failureKind` bij fouten
  - `openclaw.model_call.request_bytes`, `openclaw.model_call.response_bytes`, `openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`, `openclaw.model_call.prompt.input_messages_chars`, `openclaw.model_call.prompt.system_prompt_chars`, `openclaw.model_call.prompt.tool_definitions_count`, `openclaw.model_call.prompt.tool_definitions_chars`, `openclaw.model_call.prompt.total_chars` (alleen veilige componentgroottes, geen prompttekst)
  - `openclaw.model_call.usage.*` en `gen_ai.usage.*` wanneer het resultaat gebruiksgegevens voor die aanvraag of geaggregeerde beurt bevat
  - Span-gebeurtenis `openclaw.provider.request` met attribuut `openclaw.upstreamRequestIdHash` (begrensd, op hashes gebaseerd) wanneer het resultaat van de upstreamprovider een aanvraag-id bevat; onbewerkte id's worden nooit geëxporteerd
  - Met `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` gebruiken aanvraagspans de nieuwste GenAI-naam voor inferentiespans `{gen_ai.operation.name} {gen_ai.request.model}`. Beurtspans gebruiken `invoke_agent`, omdat OpenClaw vanuit de ondoorzichtige CLI-grens geen systeemeigen agentnaam claimt. Beide gebruiken spansoort `CLIENT` in plaats van `openclaw.model.call`.
- `openclaw.harness.run`
  - `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, `openclaw.provider`, `openclaw.model`, `openclaw.channel`
  - Bij voltooiing: `openclaw.harness.result_classification`, `openclaw.harness.yield_detected`, `openclaw.harness.items.started`, `openclaw.harness.items.completed`, `openclaw.harness.items.active`
  - Bij een fout: `openclaw.harness.phase`, `openclaw.errorCategory`, optioneel `openclaw.harness.cleanup_failed`
- `openclaw.tool.execution`
  - `gen_ai.tool.name`, `gen_ai.operation.name` (`execute_tool`), `openclaw.toolName`, `openclaw.tool.source`, optioneel `gen_ai.tool.call.id`, `openclaw.tool.owner`, `openclaw.tool.params.*`
  - Optioneel `openclaw.errorCategory`/`openclaw.errorCode` bij fouten, `openclaw.deniedReason` en `openclaw.outcome=blocked` bij weigering door beleid of sandbox
- `openclaw.exec`
  - `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`, `openclaw.exec.command_length`, `openclaw.exec.exit_code`, `openclaw.exec.exit_signal`, `openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`, `openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`, `openclaw.webhook`, `openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`, `openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`, `openclaw.ageMs`, `openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`, `openclaw.history.size`, `openclaw.context.tokens`, `openclaw.errorCategory` (geen inhoud van prompts, geschiedenis, antwoorden of sessiesleutels)
- `openclaw.tool.loop`
  - `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, optioneel `openclaw.loop.paired_tool` (geen lusberichten, parameters of tooluitvoer)
- `openclaw.memory.pressure`
  - `openclaw.memory.level`, `openclaw.memory.reason`, `openclaw.memory.rss_bytes`, `openclaw.memory.heap_used_bytes`, `openclaw.memory.heap_total_bytes`, `openclaw.memory.external_bytes`, `openclaw.memory.array_buffers_bytes`, optioneel `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms`

Wanneer inhoudsregistratie expliciet is ingeschakeld, kunnen model- en toolspans ook
begrensde, geredigeerde `openclaw.content.*`-attributen bevatten voor de specifieke
inhoudsklassen waarvoor je hebt gekozen.

## Catalogus met diagnostische gebeurtenissen

De onderstaande gebeurtenissen ondersteunen de bovenstaande metrische gegevens en spans of zijn beschikbaar voor rechtstreekse
Plugin-abonnementen. `run.progress` en `run.execution_phase` zijn uitsluitend rechtstreekse
levenscyclussignalen; de diagnostics-otel-Plugin exporteert ze niet als
zelfstandige OTLP-signalen. Gebeurtenissoorten en `run.execution_phase.phase`-waarden zijn
additief. TypeScript-consumenten moeten standaardtakken behouden in plaats van aan te nemen
dat een van beide unions permanent uitputtend is.

**Modelgebruik**

- `model.usage` - tokens, kosten, duur, context, provider/model/kanaal,
  sessie-id's. `usage` is de provider-/beurtboekhouding voor kosten en telemetrie;
  `context.used` is de huidige momentopname van prompt/context en kan lager zijn dan
  `usage.total` van de provider wanneer gecachete invoer of aanroepen binnen een toollus betrokken zijn.

**Berichtenstroom**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**Wachtrij en sessie**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase` (openbare, aan sessies gekoppelde opstartmijlpalen van de ingebedde runner)
- `diagnostic.heartbeat` (geaggregeerde tellers: webhooks/wachtrij/sessie)

**Levenscyclus van de harness**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` -
  levenscyclus per uitvoering voor de agentharness. Bevat `harnessId`, optioneel
  `pluginId`, provider/model/kanaal en uitvoerings-id. Voltooiing voegt
  `durationMs`, `outcome`, optioneel `resultClassification`, `yieldDetected`
  en `itemLifecycle`-aantallen toe. Fouten voegen `phase`
  (`prepare`/`start`/`send`/`resolve`/`cleanup`), `errorCategory` en
  optioneel `cleanupFailed` toe.

**Exec**

- `exec.process.completed` - terminalresultaat, duur, doel, modus, afsluitcode
  en fouttype. Opdrachttekst en werkmappen zijn niet
  opgenomen.
- `exec.approval.followup_suppressed` - verouderde opvolging van goedkeuring verwijderd
  nadat een sessie opnieuw was gekoppeld. Bevat `approvalId`, `reason`
  (`session_rebound`), `phase` (`direct_delivery` of `gateway_preflight`)
  en het tijdstempel van de dispatcher. Sessiesleutels, routes en opdrachttekst zijn
  niet opgenomen.

## Zonder een exporter

Houd diagnostische gebeurtenissen beschikbaar voor plugins of aangepaste sinks zonder
`diagnostics-otel` uit te voeren:

```json5
{
  diagnostics: { enabled: true },
}
```

Gebruik diagnostische vlaggen voor gerichte debuguitvoer zonder `logging.level`
te verhogen. Vlaggen zijn niet hoofdlettergevoelig en ondersteunen jokertekens
(`telegram.*` of `*`):

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

Of als een eenmalige overschrijving via een omgevingsvariabele:

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

Uitvoer van vlaggen gaat naar het standaardlogbestand (`logging.file`) en wordt nog
steeds geredigeerd door `logging.redactSensitive`. Volledige handleiding:
[Diagnostische vlaggen](/nl/diagnostics/flags).

## Uitschakelen

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

Of laat `diagnostics-otel` weg uit `plugins.allow`, of voer
`openclaw plugins disable diagnostics-otel` uit.

## Gerelateerd

- [Logboekregistratie](/nl/logging) - bestandslogboeken, console-uitvoer, volgen via de CLI en het tabblad Logs van de Control UI
- [Interne werking van Gateway-logboekregistratie](/nl/gateway/logging) - WS-logboekstijlen, subsysteemvoorvoegsels en consolevastlegging
- [Diagnostische vlaggen](/nl/diagnostics/flags) - gerichte vlaggen voor debuglogboeken
- [Diagnostische export](/nl/gateway/diagnostics) - tool voor ondersteuningsbundels voor operators (los van OTEL-export)
- [Configuratiereferentie](/nl/gateway/configuration-reference#diagnostics) - volledige referentie voor het veld `diagnostics.*`
