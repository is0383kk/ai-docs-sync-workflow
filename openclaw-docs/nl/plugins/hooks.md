---
read_when:
    - Je bouwt een plugin die before_tool_call, before_agent_reply, berichthooks of levenscyclushooks nodig heeft
    - Je moet toolaanroepen van een Plugin blokkeren, herschrijven of goedkeuring ervoor vereisen
    - Je kiest tussen interne hooks en Plugin-hooks
    - Je projecteert OpenClaw Cron-wekacties naar een externe hostplanner
summary: 'Plugin-hooks: onderschep lifecyclegebeurtenissen van agents, tools, berichten, sessies en de Gateway'
title: Plugin-hooks
x-i18n:
    generated_at: "2026-07-27T05:23:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 95d7ea2f7bfe26b5904ea3cd8f8db85ffd8163af58e03ec56d11eee992bc13d2
    source_path: plugins/hooks.md
    workflow: 16
---

Plugin-hooks zijn extensiepunten binnen het proces voor OpenClaw-plugins: inspecteer of
wijzig agentruns, toolaanroepen, berichtstromen, de sessielevenscyclus, subagent-
routering, installaties of het opstarten van de Gateway.

Gebruik in plaats daarvan [interne hooks](/nl/automation/hooks) voor een klein, door de operator geïnstalleerd
`HOOK.md`-script dat reageert op opdracht- en Gateway-gebeurtenissen zoals `/new`,
`/reset`, `/stop`, `agent:bootstrap` of `gateway:startup`.

## Snel aan de slag

Registreer getypeerde hooks met `api.on(...)` vanuit het plugin-entrypoint:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "tool-preflight",
  name: "Tool Preflight",
  register(api) {
    api.on(
      "before_tool_call",
      async (event) => {
        if (event.toolName !== "web_search") {
          return;
        }

        return {
          requireApproval: {
            title: "Webzoekopdracht uitvoeren",
            description: `Zoekopdracht toestaan: ${String(event.params.query ?? "")}`,
            severity: "info",
            timeoutMs: 60_000,
          },
        };
      },
      { priority: 50 },
    );
  },
});
```

Handlers die beslissingen of wijzigingen kunnen retourneren, worden opeenvolgend uitgevoerd in
aflopende volgorde van `priority`; handlers met dezelfde prioriteit behouden de registratievolgorde.
Handlers die alleen observeren, worden parallel uitgevoerd en fire-and-forget-observatie-
dispatches kunnen overlappen met latere gebeurtenissen. Gebruik prioriteit niet om
neveneffecten van observaties te ordenen.

`api.on(name, handler, opts?)` accepteert:

| Optie      | Effect                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `priority`  | Volgorde; hogere waarden worden eerst uitgevoerd.                                                                                                                                                                      |
| `timeoutMs` | Wachttijdbudget per hook. Wanneer dit verstrijkt, stopt OpenClaw met wachten op die handler en gaat het verder. De handler of de neveneffecten ervan worden niet geannuleerd. Laat weg om de standaardtime-out per hook van de runner te gebruiken. |

Operators kunnen hookbudgetten instellen zonder de plugincode aan te passen:

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "timeoutMs": 30000,
          "timeouts": {
            "before_prompt_build": 90000,
            "agent_end": 60000
          }
        }
      }
    }
  }
}
```

`hooks.timeouts.<hookName>` overschrijft `hooks.timeoutMs`, dat op zijn beurt de door de
plugin opgegeven waarde `api.on(..., { timeoutMs })` overschrijft. Elke waarde moet een
positief geheel getal van maximaal 600000 ms zijn. Geef voor hooks waarvan bekend is dat ze traag zijn de voorkeur aan afzonderlijke overschrijvingen,
zodat één plugin niet overal een langer budget krijgt.

Een handlerpromise waarvoor een time-out is opgetreden, blijft actief omdat hookcallbacks geen
annuleringssignaal ontvangen. De hookdispatch kan zijn Gateway-
toelating vrijgeven terwijl dat pluginwerk nog wordt uitgevoerd. Plugins die
langdurig werk beheren, moeten hun eigen annulerings- en afsluitingslevenscyclus bieden.

Uitgaande wijzigende hooks `message_sending` en `reply_payload_sending` gebruiken een
standaardduur van 15 seconden per handler. Als bij één een time-out optreedt, registreert OpenClaw de pluginfout
en gaat het verder met de meest recente payload, zodat de geserialiseerde afleveringsroute kan
worden afgerond. Stel een groter budget per hook in voor plugins die bewust trager
werk uitvoeren vóór de aflevering.

Kanaalplugins die `createReplyDispatcher` gebruiken, kunnen eveneens een groter
positief budget per fase declareren met `beforeDeliverOptions: { timeoutMs }`, of wanneer
werk wordt toegevoegd met `dispatcher.appendBeforeDeliver(handler, { timeoutMs })`.
Zonder een door de eigenaar gedeclareerd budget gebruiken deze callbacks dezelfde standaardduur van 15 seconden,
zodat een vastgelopen callback de geserialiseerde afleveringsroute niet kan vasthouden.

Elke hook ontvangt `event.context.pluginConfig`, de herleide configuratie voor de
plugin die die handler heeft geregistreerd. OpenClaw injecteert deze per handler zonder
het gedeelde gebeurtenisobject te wijzigen dat andere plugins zien.

## Hookcatalogus

Hooks zijn gegroepeerd op het oppervlak dat ze uitbreiden. **Vetgedrukte** namen accepteren een beslissings-
resultaat (blokkeren, annuleren, overschrijven of goedkeuring vereisen); de overige zijn
alleen voor observatie.

**Agentbeurt**

| Hook                            | Doel                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| `before_model_resolve`          | Provider of model overschrijven voordat sessieberichten worden geladen                                  |
| `agent_turn_prepare`            | In de wachtrij geplaatste beurtinjecties van plugins verwerken en context voor dezelfde beurt toevoegen vóór prompthooks      |
| `before_prompt_build`           | Dynamische context of systeemtekst aan de prompt toevoegen vóór de modelaanroep                          |
| **`before_agent_run`**          | De uiteindelijke prompt en sessieberichten inspecteren vóór verzending naar het model; kan de run blokkeren |
| **`before_agent_reply`**        | De modelbeurt kortsluiten met een synthetisch antwoord of stilte                           |
| **`before_agent_finalize`**     | Het natuurlijke uiteindelijke antwoord inspecteren en nog één modelpassage aanvragen                         |
| `agent_end`                     | Uiteindelijke berichten, successtatus en duur van de run observeren                                  |
| `heartbeat_prompt_contribution` | Alleen voor Heartbeat bedoelde context toevoegen voor achtergrondmonitor- en levenscyclusplugins                  |

**Gespreksobservatie**

| Hook                                      | Doel                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `model_call_started` / `model_call_ended` | Opgeschoonde metadata van provider-/modelaanroepen: timing, resultaat, begrensde hashes van aanvraag-ID's. Geen prompt- of antwoordinhoud. |
| `llm_input`                               | Providerinvoer: systeemprompt, prompt, geschiedenis                                                                     |
| `llm_output`                              | Provideruitvoer, gebruik en de herleide `contextTokenBudget` indien beschikbaar                                       |

**Tools**

| Hook                       | Doel                                                   |
| -------------------------- | --------------------------------------------------------- |
| **`before_tool_call`**     | Toolparameters herschrijven, uitvoering blokkeren of goedkeuring vereisen |
| `after_tool_call`          | Toolresultaten, fouten en duur observeren                |
| `resolve_exec_env`         | Omgevingsvariabelen die eigendom zijn van de plugin bijdragen aan `exec`   |
| **`tool_result_persist`**  | Het assistentbericht herschrijven dat uit een toolresultaat is voortgekomen |
| **`before_message_write`** | Het schrijven van een bericht dat wordt uitgevoerd inspecteren of blokkeren (zeldzaam)      |

**Berichten en aflevering**

| Hook                            | Doel                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| **`inbound_claim`**             | Een binnenkomend bericht claimen vóór agentroutering (synthetische antwoorden) |
| **`channel_pairing_requested`** | Nieuw aangemaakte DM-koppelingsverzoeken observeren                         |
| `message_received`              | Binnenkomende inhoud, afzender, thread en metadata observeren             |
| **`message_sending`**           | Uitgaande inhoud herschrijven of aflevering annuleren                       |
| **`reply_payload_sending`**     | Genormaliseerde antwoordpayloads vóór aflevering wijzigen of annuleren        |
| `message_sent`                  | Geslaagde of mislukte uitgaande aflevering observeren                      |
| **`before_dispatch`**           | Een uitgaande dispatch inspecteren of herschrijven vóór overdracht aan het kanaal    |
| **`reply_dispatch`**            | Deelnemen aan de uiteindelijke pipeline voor antwoorddispatch                  |

**Sessies en Compaction**

| Hook                                     | Doel                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `session_start` / `session_end`          | Grenzen van de sessielevenscyclus volgen. `reason` is een van `new`, `reset`, `idle`, `daily`, `compaction`, `deleted`, `shutdown`, `restart` of `unknown`. `shutdown`/`restart` worden geactiveerd vanuit de afsluitfinalizer van de Gateway wanneer het proces stopt of opnieuw start met actieve sessies, zodat plugins (geheugen, transcriptopslagplaatsen) spookrijen kunnen voltooien in plaats van ze tussen herstarts geopend te laten. De finalizer is begrensd, zodat een trage plugin SIGTERM/SIGINT niet kan blokkeren. |
| `before_compaction` / `after_compaction` | Compaction-cycli observeren of annoteren                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `before_reset`                           | Gebeurtenissen voor het opnieuw instellen van sessies observeren (`/reset`, programmatische resets)                                                                                                                                                                                                                                                                                                                                                                                                     |

Voor `sessions.create`-aanroepen met `parentSessionKey` en `emitCommandHooks: true` ontvangt een afzonderlijk kind altijd `session_start`. Aanroepers declareren met `succeedsParent` of de ouder ook een terminale `session_end` ontvangt: `true` betekent opvolger, `false` betekent parallel kind. Weglaten behoudt het verouderde gedrag waarbij de ouder wordt doorgeschoven. De hooks `command:new` en `before_reset` beschrijven in beide gevallen nog steeds de aangevraagde actie `/new`.

**Subagents**

- `subagent_spawned` / `subagent_ended` - observeer het starten en voltooien van de subagent.
- `subagent_delivery_target` - compatibiliteitshook voor het afleveren van voltooiingen wanneer geen kernsessiebinding een route kan projecteren.
- `subagent_spawning` - verouderde compatibiliteitshook. De kern bereidt nu `thread: true`-subagentbindingen voor via adapters voor kanaalsessiebindingen voordat `subagent_spawned` wordt geactiveerd.
- `subagent_spawned` bevat `resolvedModel` en `resolvedProvider` wanneer OpenClaw vóór het starten het native model van de onderliggende sessie heeft vastgesteld.
- `subagent_ended` bevat `targetSessionKey` (identiteit - komt overeen met `subagent_spawned.childSessionKey`), `targetKind` (`"subagent"` of `"acp"`), `reason`, optioneel `outcome` (`"ok"`, `"error"`, `"timeout"`, `"killed"`, `"reset"` of `"deleted"`), optioneel `error`, `runId`, `endedAt`, `accountId` en `sendFarewell`. Het bevat **niet** `agentId` of `childSessionKey`; gebruik `targetSessionKey` om dit te correleren met de bijbehorende `subagent_spawned`-gebeurtenis.

**Levenscyclus**

| Hook                             | Doel                                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `gateway_start` / `gateway_stop` | Door plugins beheerde services samen met de Gateway starten of stoppen                               |
| `deactivate`                     | Verouderde compatibiliteitsalias voor `gateway_stop`; gebruik `gateway_stop` in nieuwe plugins      |
| `cron_reconciled`                | Na het starten of opnieuw laden afstemmen op de volledige Cron-status van de Gateway                  |
| `cron_changed`                   | Wijzigingen in de door de Gateway beheerde Cron-levenscyclus observeren (toegevoegd, bijgewerkt, verwijderd, gestart, voltooid, gepland) |
| **`before_install`**             | Klaargezet installatie­materiaal voor Skills of plugins inspecteren vanuit een geladen pluginruntime |

### Verzoeken voor kanaalkoppeling

Gebruik `channel_pairing_requested` wanneer een plugin een operator moet waarschuwen of
een auditrecord moet schrijven nadat een niet-gekoppelde afzender van een privébericht een openstaand
koppelingsverzoek heeft aangemaakt. De hook wordt aangeroepen wanneer het verzoek wordt aangemaakt; de kanaalaflevering van
het koppelingsantwoord wordt niet vertraagd door trage of falende hookhandlers.

```typescript
api.on("channel_pairing_requested", async (event) => {
  await notifyOperator({
    text: `Nieuw ${event.channel}-koppelingsverzoek van ${event.senderId}: ${event.code}`,
  });
});
```

De hook dient alleen voor observatie. Deze keurt het koppelingsantwoord niet goed, wijst het niet af, onderdrukt het niet en herschrijft
het niet. De payload bevat het kanaal, optioneel `accountId`,
kanaalgebonden `senderId`, koppelings-`code` en kanaalmetadata. Behandel de
koppelingscode als een actief, eenmalig goedkeuringscredential en lever deze alleen af bij een
vertrouwde operatorbestemming. Behandel `metadata` als niet-vertrouwde, door de afzender aangeleverde identiteitstekst.
De hook bevat niet de inhoud of media van het inkomende bericht.

## Hooks voor runtimefoutopsporing

Gebruik `before_model_resolve` om voor een agentbeurt van provider of model te wisselen - deze
wordt uitgevoerd voordat het model wordt vastgesteld. `llm_output` wordt pas uitgevoerd nadat een modelpoging
assistentuitvoer heeft geproduceerd.

Inspecteer runtimeregistraties voor bewijs van het effectieve sessiemodel en
gebruik vervolgens `openclaw sessions` of de sessie-/statusoppervlakken van de Gateway. Start de Gateway met
`--raw-stream` en `--raw-stream-path <path>` om providerpayloads te debuggen en
onbewerkte modelstreamgebeurtenissen naar een jsonl-bestand te schrijven.

## Beleid voor toolaanroepen

`before_tool_call` ontvangt:

- `event.toolName`
- `event.params`
- optioneel `event.toolKind` en `event.toolInputKind`, door de host bepaalde
  onderscheidende waarden voor tools die bewust dezelfde naam gebruiken; zo gebruiken buitenste
  `exec`-aanroepen in codemodus bijvoorbeeld `toolKind: "code_mode_exec"` en bevatten ze
  `toolInputKind: "javascript" | "typescript"` wanneer de invoertaal
  bekend is
- optioneel `event.derivedPaths`, zo goed mogelijk door de host afgeleide hints voor doelpaden
  voor bekende toolenveloppen zoals `apply_patch`; deze paden kunnen
  onvolledig zijn of te ruim inschatten wat de tool daadwerkelijk zal wijzigen (bijvoorbeeld
  bij ongeldige of gedeeltelijke invoer)
- optioneel `event.runId`
- optioneel `event.toolCallId`
- contextvelden zoals `ctx.agentId`, `ctx.sessionKey`, `ctx.sessionId`,
  `ctx.runId`, `ctx.toolKind`, `ctx.toolInputKind` en diagnostische `ctx.trace`
- optioneel `ctx.requester`, de door de host afgeleide aanvrager die de huidige
  berichtuitvoering heeft geïnitieerd. Deze kan `channel`, `accountId`, `senderId`,
  `senderIsOwner` en provider-native `roleIds` bevatten. Ontbrekende velden zijn niet bewezen,
  en bieden geen valse zekerheid; werk strikt gesloten wanneer het beleid deze vereist.

Deze kan het volgende retourneren:

```typescript
type BeforeToolCallResult = {
  params?: Record<string, unknown>;
  block?: boolean;
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number;
    /** @deprecated Niet-afgehandelde goedkeuringen leiden altijd tot weigering. */
    timeoutBehavior?: "allow" | "deny";
    allowedDecisions?: Array<"allow-once" | "allow-always" | "deny">;
    pluginId?: string;
    onResolution?: (
      decision: "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled",
    ) => Promise<void> | void;
  };
};
```

Beveiligingsgedrag voor getypeerde levenscyclushooks:

- `block: true` is definitief en slaat handlers met een lagere prioriteit over.
- `block: false` wordt behandeld alsof er geen beslissing is genomen.
- `params` herschrijft de toolparameters voor uitvoering.
- `requireApproval` pauzeert de agentuitvoering en vraagt de gebruiker om toestemming via plugin-
  goedkeuringen. `/approve` kan zowel uitvoerings- als plugingoedkeuringen verlenen. Bij native `PreToolUse`-doorgiften
  in de rapportagemodus van de Codex-appserver wordt dit overgelaten aan het
  bijbehorende goedkeuringsverzoek van de appserver; zie
  [Codex-harnessruntime](/nl/plugins/codex-harness-runtime#hook-boundaries).
- Een `block: true` met een lagere prioriteit kan nog steeds blokkeren nadat een hook met een hogere prioriteit
  om goedkeuring heeft gevraagd.
- `onResolution` ontvangt de vastgestelde beslissing: `allow-once`, `allow-always`,
  `deny`, `timeout` of `cancelled`.

### Afzenderbewust beleid in één bestand

Een zelfstandig pluginbestand kan implementatiespecifiek beleid in code bewaren
in plaats van nog een configuratieschema toe te voegen. Dit voorbeeld geeft eigenaren toegang tot elke tool,
laat geconfigureerde beheerders een conservatieve set tools en berichtacties gebruiken,
en stelt `/fix` beschikbaar aan afzenders die al door de kanaalconfiguratie zijn geautoriseerd:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const AGENT_ID = "maintenance-agent";
const MAINTAINER_SCOPES = [
  {
    channel: "discord",
    accountId: "operations",
    senderIds: new Set(["maintainer-user-id"]),
    roleIds: new Set(["maintainer-role-id"]),
  },
];
const MAINTAINER_TOOLS = new Set(["read", "web_fetch", "web_search", "session_status", "message"]);
const MAINTAINER_MESSAGE_ACTIONS = new Set(["react", "reply", "thread-create", "thread-reply"]);

export default definePluginEntry({
  id: "maintenance-access",
  name: "Onderhoudstoegang",
  description: "Pas afzenderbewust toolbeleid toe op de onderhoudsagent.",
  register(api) {
    api.on("before_tool_call", (event, ctx) => {
      if (ctx.agentId !== AGENT_ID) {
        return;
      }

      const requester = ctx.requester;
      if (requester?.senderIsOwner === true) {
        return;
      }

      const maintainerScope = requester
        ? MAINTAINER_SCOPES.find(
            (scope) =>
              scope.channel === requester.channel && scope.accountId === requester.accountId,
          )
        : undefined;
      const isMaintainer =
        maintainerScope !== undefined &&
        ((requester?.senderId !== undefined && maintainerScope.senderIds.has(requester.senderId)) ||
          requester?.roleIds?.some((roleId) => maintainerScope.roleIds.has(roleId)) === true);
      if (!isMaintainer) {
        return { block: true, blockReason: "Beheerderstoegang vereist." };
      }

      if (event.toolName === "message") {
        const action = typeof event.params.action === "string" ? event.params.action : "";
        if (MAINTAINER_MESSAGE_ACTIONS.has(action)) {
          return;
        }
        return { block: true, blockReason: `Eigenaar vereist voor message.${action || "unknown"}.` };
      }

      if (MAINTAINER_TOOLS.has(event.toolName)) {
        return;
      }
      return { block: true, blockReason: `Eigenaar vereist voor ${event.toolName}.` };
    });

    api.registerCommand({
      name: "fix",
      description: "Vraag de onderhoudsagent een probleem te onderzoeken en op te lossen.",
      acceptsArgs: true,
      requireAuth: true,
      handler: async (ctx) =>
        ctx.agentId === AGENT_ID
          ? { continueAgent: true }
          : { text: "Deze opdracht is alleen beschikbaar in het onderhoudsgesprek." },
    });
  },
});
```

Laad het bestand rechtstreeks en start de Gateway opnieuw:

```json5
{
  agents: {
    list: [
      {
        id: "maintenance-agent",
        workspace: "~/.openclaw/workspace-maintenance",
      },
    ],
  },
  bindings: [
    {
      agentId: "maintenance-agent",
      match: {
        channel: "discord",
        accountId: "operations",
        peer: { kind: "channel", id: "maintenance-channel-id" },
      },
    },
  ],
  plugins: {
    load: { paths: ["~/.openclaw/policies/maintenance-access.ts"] },
  },
}
```

`AGENT_ID` moet de agent benoemen die aan het onderhoudsgesprek is gebonden. De
binding selecteert die agent voor normale berichten en `/fix`; het zelfstandige bestand
blijft de enige eigenaar van het toolbeleid voor eigenaren tegenover beheerders.

`requireAuth: true` hergebruikt de bestaande afzendertoelating van elk kanaal. Voor
Discord kan een `users`-/`roles`-toelatingslijst van een server of kanaal de
onderhoudsdoelgroep autoriseren. Andere kanalen kunnen stabiele afzender-id's gebruiken. De hook past vervolgens
de verfijndere beslissing per tool toe op elke toolaanroep tijdens de uitvoering, inclusief
native `PreToolUse`-aanroepen van Codex. Deze kan een tool die het model ziet blokkeren, maar kan
geen door de host weggelaten tool toevoegen. Bestaand sandbox-, uitvoeringsgoedkeurings-, alleen-voor-eigenaren-
kerntool- en kanaalbeleid blijft van toepassing; de hook kan deze beperkingen niet omzeilen.

Beperk afzender- en rol-id's tot een exact kanaal-/accountpaar zoals weergegeven; beide zijn
providerlokale naamruimten. Houd de toelatingslijsten conservatief. Voeg alleen schrijf-
of uitvoeringstools toe wanneer het sandbox- en goedkeuringsbeleid van de implementatie
dat veilig maakt. Bepaal voor geautomatiseerde of systeemuitvoeringen expliciet of een ontbrekende
`ctx.requester` mag worden doorgelaten; het voorbeeld weigert deze voor de betrokken agent.

Zie [Verzoeken om pluginmachtigingen](/nl/plugins/plugin-permission-requests) voor
goedkeuringsroutering, beslissingsgedrag en wanneer je `requireApproval` moet gebruiken in plaats
van optionele tools of uitvoeringsgoedkeuringen.

Plugins die beleid op hostniveau nodig hebben, kunnen vertrouwd toolbeleid registreren met
`api.registerTrustedToolPolicy(...)`. Dit wordt uitgevoerd vóór gewone
`before_tool_call`-hooks en vóór normale hookbeslissingen. Gebundeld vertrouwd
beleid wordt eerst uitgevoerd; vertrouwd beleid van geïnstalleerde plugins volgt daarna in de laadvolgorde
van plugins; gewone `before_tool_call`-hooks worden daarna uitgevoerd. Gebundelde plugins behouden
het bestaande pad voor vertrouwd beleid. Geïnstalleerde plugins moeten expliciet zijn ingeschakeld
en elke beleids-id declareren in `contracts.trustedToolPolicies`; niet-gedeclareerde id's
worden vóór registratie geweigerd. Beleids-id's zijn beperkt tot de registrerende
plugin, zodat verschillende plugins dezelfde lokale id mogen hergebruiken. Gebruik dit niveau alleen
voor door de host vertrouwde controles, zoals werkruimtebeleid, budgethandhaving of
veiligheid van gereserveerde workflows.

### Hook voor exec-omgeving

`resolve_exec_env` laat plugins omgevingsvariabelen toevoegen aan `exec`-
toolaanroepen voordat de opdracht wordt uitgevoerd. De hook ontvangt:

- `event.sessionKey`
- `event.toolName`, momenteel altijd `"exec"`
- `event.host`, een van `"gateway"`, `"sandbox"` of `"node"`
- contextvelden zoals `ctx.agentId`, `ctx.sessionKey`,
  `ctx.messageProvider` en `ctx.channelId`

Retourneer een `Record<string, string>` om deze met de exec-omgeving samen te voegen. Handlers
worden in prioriteitsvolgorde uitgevoerd; latere resultaten overschrijven eerdere
resultaten voor dezelfde sleutel.

Hookuitvoer wordt vóór het samenvoegen gefilterd volgens het sleutelbeleid voor
de exec-omgeving van de host. `PATH` wordt altijd verwijderd (opdrachtresolutie en
controles op veilige binaire bestanden zijn ervan afhankelijk). Ongeldige sleutels en gevaarlijke
overschrijvingssleutels voor de host, zoals `LD_*`, `DYLD_*`, `NODE_OPTIONS`,
proxyvariabelen (`HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`)
en TLS-overschrijvingsvariabelen (`NODE_TLS_REJECT_UNAUTHORIZED`, `SSL_CERT_FILE` en vergelijkbare)
worden verwijderd. De gefilterde pluginomgeving wordt opgenomen in de goedkeurings-/
auditmetadata van de Gateway en doorgestuurd naar uitvoeringsaanvragen voor de node-host.

### Persistentie van toolresultaten

Toolresultaten kunnen gestructureerde `details` bevatten voor UI-weergave, diagnostiek,
mediaroutering of metadata die eigendom is van een plugin. Behandel `details` als runtimemetadata,
niet als promptinhoud:

- OpenClaw verwijdert `toolResult.details` vóór herhaling door de provider en invoer voor
  Compaction, zodat metadata geen modelcontext wordt.
- Gepersisteerde sessie-items behouden alleen begrensde `details`. Te grote details worden
  vervangen door een compacte samenvatting en `persistedDetailsTruncated: true`.
- `tool_result_persist` en `before_message_write` worden uitgevoerd vóór de uiteindelijke
  persistentielimiet. Houd geretourneerde `details` klein en plaats
  promptrelevante tekst niet uitsluitend in `details`; plaats voor het model zichtbare tooluitvoer in
  `content`.

## Hooks voor prompts en modellen

Gebruik de fasespecifieke hooks voor nieuwe plugins:

- `before_model_resolve`: ontvangt alleen de huidige prompt en metadata van
  bijlagen. Retourneer `providerOverride` of `modelOverride`.
- `agent_turn_prepare`: ontvangt de huidige prompt, voorbereide
  sessieberichten en eventuele precies-éénmaal-injecties uit de wachtrij die voor deze sessie zijn opgehaald.
  Retourneer `prependContext` of `appendContext`.
- `before_prompt_build`: ontvangt de huidige prompt en sessieberichten.
  Retourneer `prependContext`, `appendContext`, `systemPrompt`,
  `prependSystemContext` of `appendSystemContext`.
- `heartbeat_prompt_contribution`: wordt alleen uitgevoerd voor Heartbeat-beurten en retourneert
  `prependContext` of `appendContext`. Bedoeld voor achtergrondmonitors die
  de huidige status moeten samenvatten zonder door de gebruiker geïnitieerde beurten te wijzigen.

`before_agent_run` wordt uitgevoerd na het samenstellen van de prompt en vóór enige modelinvoer,
inclusief het laden van promptlokale afbeeldingen en de observatie `llm_input`. De hook ontvangt
de huidige gebruikersinvoer als `prompt`, plus de geladen sessiegeschiedenis in `messages`
en de actieve systeemprompt. Retourneer `{ outcome: "block", reason, message? }`
om de uitvoering te stoppen voordat het model de prompt leest. `reason` is intern;
`message` is de vervanging die aan de gebruiker wordt getoond. Alleen de uitkomsten `pass` en `block`
worden ondersteund; niet-ondersteunde beslissingsvormen worden standaard geweigerd.

Wanneer een uitvoering wordt geblokkeerd, slaat OpenClaw alleen de vervangende tekst op in
`message.content`, samen met niet-gevoelige blokkeringsmetadata, zoals de id van de
blokkerende plugin en het tijdstip. De oorspronkelijke gebruikerstekst wordt niet bewaard in het transcript
of in toekomstige context. Interne blokkeringsredenen worden als gevoelig behandeld en
uitgesloten van transcript-, geschiedenis-, broadcast-, log- en diagnostiekpayloads.
Voor observeerbaarheid moeten opgeschoonde velden worden gebruikt, zoals de id van de blokkering, de uitkomst,
het tijdstip of een veilige categorie.

Hooks voor agentbeurten, waaronder `agent_end`, bevatten `event.runId` wanneer OpenClaw
de actieve uitvoering kan identificeren; dezelfde waarde staat ook in `ctx.runId`. Door Cron aangestuurde
uitvoeringen stellen ook `ctx.jobId` (de id van de oorspronkelijke Cron-taak) beschikbaar in de context
van de agentbeurt, zodat hooks metrieken, neveneffecten of status kunnen beperken tot een specifieke
geplande taak. `ctx.jobId` maakt geen deel uit van de toolcontext `before_tool_call`.

Voor uitvoeringen die vanuit een kanaal afkomstig zijn, identificeren `ctx.channel` en `ctx.messageProvider`
het provideroppervlak, zoals `discord` of `telegram`, terwijl `ctx.channelId`
de doelaanduiding van het gesprek is wanneer OpenClaw die kan afleiden uit de
sessiesleutel of afleveringsmetadata.

Wanneer de identiteit van de afzender beschikbaar is, bevatten contexten van agenthooks ook:

- `ctx.senderId` - kanaalspecifieke afzender-id (bijvoorbeeld Feishu `open_id`, Discord-
  gebruikers-id). Ingevuld wanneer de uitvoering afkomstig is van een gebruikersbericht met bekende
  afzendermetadata.
- `ctx.chatId` - transport-eigen gespreksidentificatie (bijvoorbeeld Feishu
  `chat_id`, Telegram `chat_id`). Ingevuld wanneer het oorspronkelijke kanaal
  een eigen gespreks-id levert.
- `ctx.channelContext.sender.id` - dezelfde afzender-id als `ctx.senderId`, onder
  een object dat eigendom is van het kanaal en dat plugins kunnen uitbreiden met kanaalspecifieke velden.
- `ctx.channelContext.chat.id` - dezelfde gespreks-id als `ctx.chatId`,
  onder een object dat eigendom is van het kanaal en dat plugins kunnen uitbreiden met kanaalspecifieke
  velden.

De kern definieert alleen de geneste velden van `id`. Kanaalplugins die uitgebreidere
afzender- of chatmetadata via de inkomende helper doorgeven, kunnen
`PluginHookChannelSenderContext` of `PluginHookChannelChatContext` uitbreiden vanuit
`openclaw/plugin-sdk/channel-inbound`:

```ts
declare module "openclaw/plugin-sdk/channel-inbound" {
  interface PluginHookChannelSenderContext {
    unionId?: string;
    userId?: string;
  }
}
```

Kanaalplugins geven deze velden door via de inkomende SDK-helper:

```ts
buildChannelInboundEventContext({
  // ...
  channelContext: {
    sender: { id: senderOpenId, unionId, userId },
    chat: { id: chatId },
  },
});
```

Deze velden zijn optioneel en ontbreken bij uitvoeringen die vanuit het systeem afkomstig zijn (Heartbeat,
Cron, exec-gebeurtenis).

`ctx.senderExternalId` blijft beschikbaar als verouderd veld voor broncompatibiliteit voor
oudere plugins. De kern vult dit niet in; nieuwe kanaalspecifieke
afzenderidentiteiten horen onder `ctx.channelContext.sender` via module-
uitbreiding.

`agent_end` is een observatiehook. Gateway- en persistente harnaspaden voeren
deze na de beurt asynchroon zonder erop te wachten uit, terwijl kortlevende eenmalige CLI-paden wachten
op de hookbelofte voordat het proces wordt opgeschoond, zodat vertrouwde plugins
terminale observeerbaarheidsgegevens kunnen wegschrijven of status kunnen vastleggen. De hookrunner past een time-out van 30 seconden
toe, zodat een vastgelopen plugin of insluitingsendpoint de hookbelofte niet
voor altijd in behandeling kan laten. Een time-out wordt gelogd en OpenClaw gaat door; het
annuleert netwerkwerk dat eigendom is van de plugin niet, tenzij de plugin ook een eigen afbreek-
signaal gebruikt.

Gebruik `model_call_started` en `model_call_ended` voor telemetrie van provideraanroepen
die geen onbewerkte prompts, geschiedenis, antwoorden, headers, aanvraag-
body's of provideraanvraag-id's mag ontvangen. Deze hooks bevatten stabiele metadata zoals
`runId`, `callId`, `provider`, `model`, optioneel `api`/`transport`, terminale
`durationMs`/`outcome` en `upstreamRequestIdHash` wanneer OpenClaw een
begrensde hash van de provideraanvraag-id kan afleiden. Wanneer de runtime metadata voor het
contextvenster heeft vastgesteld, bevatten de hookgebeurtenis en context ook
`contextTokenBudget`, het effectieve tokenbudget na limieten van model/configuratie/agent,
plus `contextWindowSource` en `contextWindowReferenceTokens` wanneer een
lagere limiet is toegepast.

`before_agent_finalize` wordt alleen uitgevoerd wanneer een harnas op het punt staat een natuurlijk
definitief assistentantwoord te accepteren. Dit is niet het annuleringspad `/stop` en wordt niet
uitgevoerd wanneer de gebruiker een beurt afbreekt. Retourneer `{ action: "revise", reason }` om
het harnas vóór afronding om nog één modeldoorgang te vragen, `{ action:
"finalize", reason? }` om afronding af te dwingen, of laat een resultaat weg om door te gaan.
Handlers hebben standaard een budget van 15s; bij een time-out logt OpenClaw de fout en
gaat het verder met het oorspronkelijke definitieve antwoord.
Native `Stop`-hooks van Codex worden naar deze hook doorgestuurd als
`before_agent_finalize`-beslissingen van OpenClaw.

Bij het retourneren van `action: "revise"` kunnen plugins `retry`-metadata opnemen om
de extra modeldoorgang begrensd en veilig voor herhaling te maken:

```typescript
type BeforeAgentFinalizeRetry = {
  instruction: string;
  idempotencyKey?: string;
  maxAttempts?: number;
};
```

`instruction` wordt toegevoegd aan de revisiereden die naar het harnas wordt gestuurd.
Met `idempotencyKey` kan de host nieuwe pogingen voor hetzelfde pluginverzoek
over equivalente afrondingsbeslissingen heen tellen, en `maxAttempts` begrenst hoeveel extra
doorgangen de host toestaat voordat deze doorgaat met het natuurlijke definitieve antwoord.

Niet-gebundelde plugins die onbewerkte gesprekshooks nodig hebben (`before_model_resolve`,
`before_agent_reply`, `llm_input`, `llm_output`, `before_agent_finalize`,
`agent_end` of `before_agent_run`) moeten het volgende instellen:

```json
{
  "plugins": {
    "entries": {
      "my-plugin": {
        "hooks": {
          "allowConversationAccess": true
        }
      }
    }
  }
}
```

Promptwijzigende hooks en duurzame injecties voor de volgende beurt kunnen per
plugin worden uitgeschakeld met `plugins.entries.<id>.hooks.allowPromptInjection=false`.

### Sessie-uitbreidingen en injecties voor de volgende beurt

Workflowplugins kunnen een kleine JSON-compatibele sessiestatus persistent opslaan met
`api.session.state.registerSessionExtension(...)` en deze bijwerken via de
Gateway-methode `sessions.pluginPatch`. Sessierijen projecteren geregistreerde
uitbreidingsstatus via `pluginExtensions`, zodat de Control UI en andere
clients door plugins beheerde status kunnen weergeven zonder de interne werking van plugins te kennen.
`api.registerSessionExtension(...)` werkt nog steeds, maar is verouderd ten gunste van
de naamruimte `api.session.state`.

Gebruik `api.session.workflow.enqueueNextTurnInjection(...)` wanneer een plugin
duurzame context precies één keer aan de volgende modelbeurt moet doorgeven (de `api.enqueueNextTurnInjection(...)` op het hoogste niveau is een verouderde alias met hetzelfde
gedrag). OpenClaw haalt injecties uit de wachtrij vóór prompthooks, verwijdert
verlopen injecties en dedupliceert per plugin op basis van `idempotencyKey`. Dit is
het juiste koppelvlak voor hervattingen na goedkeuring, beleidssamenvattingen, delta's van achtergrondmonitors
en voortzettingen van opdrachten die bij de volgende beurt zichtbaar moeten zijn voor het model,
maar geen permanente tekst in de systeemprompt mogen worden.

Opschoonsemantiek maakt deel uit van het contract. Opschooncallbacks voor sessie-uitbreidingen en
de runtimelevenscyclus ontvangen `reset`, `delete`, `disable` of
`restart`. De host verwijdert de persistente sessie-uitbreidingsstatus
en wachtende injecties voor de volgende beurt van de betreffende plugin bij reset/verwijdering/uitschakeling; bij herstart
blijft duurzame sessiestatus behouden, terwijl opschooncallbacks plugins scheduler-taken,
uitvoeringscontext en andere resources buiten de normale gegevensstroom van de oude
runtimegeneratie laten vrijgeven.

## Berichthooks

Gebruik berichthooks voor routerings- en afleveringsbeleid op kanaalniveau:

- `message_received`: observeer inkomende inhoud, afzender, `threadId`,
  `messageId`, `senderId`, optionele correlatie van uitvoering/sessie, geordende `media`
  en metadata.
- `message_sending`: herschrijf `content` of retourneer `{ cancel: true }`.
- `reply_payload_sending`: herschrijf genormaliseerde `ReplyPayload`-objecten
  (inclusief `presentation`, `delivery`, mediaverwijzingen en tekst) of retourneer
  `{ cancel: true }`.
- `message_sent`: observeer het uiteindelijke succes of de uiteindelijke mislukking.

Voor TTS-antwoorden met alleen audio kan `content` het verborgen gesproken
transcript bevatten, zelfs wanneer de kanaalpayload geen zichtbare tekst/bijschrift bevat.
Het herschrijven van die `content` werkt alleen het voor de hook zichtbare transcript bij; het wordt niet
weergegeven als mediabijschrift.

`reply_payload_sending`-gebeurtenissen kunnen `usageState` bevatten, een naar beste vermogen gemaakte live
momentopname per beurt van model/gebruik/context. Duurzame aflevering, herstelde herhaling en
antwoorden zonder exacte uitvoeringscorrelatie laten dit weg.

Hookcontexten voor berichten stellen, indien beschikbaar, stabiele correlatievelden beschikbaar:
`ctx.sessionKey`, `ctx.runId`, `ctx.messageId`, `ctx.senderId`, `ctx.trace`,
`ctx.traceId`, `ctx.spanId`, `ctx.parentSpanId` en `ctx.callDepth`. Contexten voor inkomende
berichten en `before_dispatch` stellen ook antwoordmetadata beschikbaar wanneer het kanaal
op zichtbaarheid gefilterde gegevens van geciteerde berichten bevat: `replyToId`, `replyToIdFull`,
`replyToBody`, `replyToSender` en `replyToIsQuote`. Geef de voorkeur aan deze
eersteklasvelden voordat je verouderde metadata leest.

Geef de voorkeur aan getypeerde velden `threadId` en `replyToId` voordat je kanaalspecifieke
metadata gebruikt.

Gebeurtenissen voor inkomende claims en ontvangen berichten stellen `media?:
PluginHookMediaFact[]` beschikbaar als de canonieke API voor bijlagen. Elk gegeven kan
`path`, `url`, `contentType`, `kind`, `transcribed`, `messageId` en
`workspaceDir` bevatten; de positie in de array vormt de identiteit van de bijlage. Wanneer een externe bijlage
nog niet lokaal is klaargezet, wordt `media` weggelaten,
`mediaStagingPending: true` en bevat `originalMedia` de gegevens
van de provider. Beschouw `originalMedia.path` niet als lokaal leesbaar totdat een latere
klaargezette gebeurtenis `media` aanlevert.

De enkelvoudige/meervoudige `mediaPath`, `mediaUrl`, `mediaType`, `mediaPaths`,
`mediaUrls`, `mediaTypes` en overeenkomende metadata-eigenschappen van `originalMedia*` zijn
verouderde compatibiliteitsaliassen. Nieuwe hooks moeten de getypeerde arrays op het hoogste niveau
gebruiken.

Beslissingsregels:

- `message_sending` met `cancel: true` is definitief.
- `message_sending` met `cancel: false` wordt behandeld als geen beslissing.
- Herschreven `content` gaat door naar hooks met een lagere prioriteit, tenzij een latere hook
  de bezorging annuleert.
- `reply_payload_sending` wordt uitgevoerd na normalisatie van de payload en vóór bezorging
  via het kanaal, inclusief antwoorden die worden teruggestuurd naar het oorspronkelijke kanaal.
  Handlers worden opeenvolgend uitgevoerd en elke handler ziet de nieuwste payload die
  door handlers met een hogere prioriteit is geproduceerd.
- `reply_payload_sending`-payloads stellen geen runtimevertrouwensmarkeringen beschikbaar, zoals
  `trustedLocalMedia`; plugins kunnen de vorm van de payload bewerken, maar kunnen geen vertrouwen voor lokale
  media verlenen.
- `message_sending` kan `cancelReason` en begrensde `metadata` retourneren bij een
  annulering. Nieuwe API's voor de berichtlevenscyclus stellen dit beschikbaar als een onderdrukt
  bezorgresultaat met reden `cancelled_by_message_sending_hook`; verouderde
  directe bezorging blijft voor compatibiliteit een lege resultaatarray retourneren.
- `message_sent` dient alleen voor observatie. Fouten in handlers worden gelogd en wijzigen
  het bezorgresultaat niet.

## Installatiehooks

Gebruik `security.installPolicy` voor door de operator beheerde toestaan/blokkeren-beslissingen. Dat
beleid wordt uitgevoerd vanuit de OpenClaw-configuratie, geldt voor installatie- en updatepaden via de CLI en
weigert bij fouten wanneer het is ingeschakeld maar niet beschikbaar is.

`before_install` is een levenscyclus-hook van de pluginruntime. Deze wordt na
`security.installPolicy` alleen uitgevoerd in het OpenClaw-proces waarin pluginhooks
al zijn geladen, zoals installatieflows via een Gateway. Deze is nuttig voor
observaties, waarschuwingen en compatibiliteitscontroles die eigendom zijn van plugins, maar vormt niet
de primaire beveiligingsgrens voor ondernemingen of hosts bij installaties. Het veld
`builtinScan` blijft voor compatibiliteit aanwezig in de gebeurtenispayload, maar
OpenClaw voert geen ingebouwde blokkering van gevaarlijke code tijdens de installatie meer uit, dus het
is een leeg `ok`-resultaat. Retourneer aanvullende bevindingen of
`{ block: true, blockReason }` om de installatie in dat proces te stoppen.

`block: true` is definitief. `block: false` wordt behandeld als geen beslissing. Fouten in handlers
blokkeren de installatie volgens het fail-closed-principe.

## Levenscyclus van de Gateway

Gebruik `gateway_start` om algemene pluginservices te starten en `gateway_stop` om
langlopende resources op te ruimen. De Cron-planner kan nog bezig zijn met laden wanneer
`gateway_start` wordt uitgevoerd, dus gebruik deze niet als basissignaal voor een externe
Cron-projectie.

Vertrouw voor runtimeservices die eigendom zijn van plugins niet op de interne hook
`gateway:startup`.

`cron_reconciled` wordt geactiveerd nadat de Cron-planner van de Gateway en de bijbehorende
watchers bij afsluiten hun duurzame status hebben gereconcilieerd. Dit gebeurt zowel bij de eerste
start als bij vervanging van de planner tijdens het opnieuw laden van de configuratie. De gebeurtenis rapporteert
`reason` (`startup` of `reload`) en de effectieve status `enabled`. Uitgeschakelde
Cron activeert de gebeurtenis nog steeds met `enabled: false`, zodat een externe projectie
verouderde wekacties kan wissen. Gebruik `ctx.getCron?.()` voor de exacte plannerinstantie die
de reconciliatie heeft voltooid; een latere herlaadactie wijst die callback niet opnieuw toe.
`ctx.abortSignal` is eigenaar van dezelfde momentopname van de planner. De Gateway breekt deze af zodra
een nieuwere planner wordt geactiveerd of het afsluiten begint. Geef deze door aan elk
duurzaam neveneffect en accepteer de momentopname niet nadat deze is afgebroken.
Dit is een levenscyclussignaal van de planner, geen activeringssignaal van een plugin:
een hot reload van alleen een plugin speelt dit niet opnieuw af. Een nieuw ingeschakelde afnemer ontvangt
zijn eerste basisstatus bij de volgende vervanging van de planner of start van de Gateway.

Net als andere observatiehooks kunnen callbacks van `gateway_start` en `cron_reconciled`
overlappen. Als beide handlers dezelfde plugininitialisatie delen, coördineer ze dan
met een pluginlokale gereedheidsbelofte in plaats van afhankelijk te zijn van de callbackvolgorde.

`cron_changed` wordt geactiveerd voor Cron-levenscyclusgebeurtenissen die eigendom zijn van de Gateway, met een getypeerde
gebeurtenispayload voor de redenen `added`, `updated`, `removed`, `started`, `finished`
en `scheduled`. De gebeurtenis bevat een momentopname van `PluginHookGatewayCronJob`
(inclusief `state.nextRunAtMs`, `state.lastRunStatus` en
`state.lastError` wanneer aanwezig) plus een `PluginHookGatewayCronDeliveryStatus`
van `not-requested` | `delivered` | `not-delivered` | `unknown`. Verwijderingsgebeurtenissen
vinden na de commit plaats: ze worden alleen geactiveerd nadat duurzame verwijdering is geslaagd en bevatten nog steeds
de momentopname van de verwijderde taak, zodat externe planners de status kunnen reconciliëren.

Een `scheduled`-gebeurtenis vindt na de commit plaats: deze wordt alleen geactiveerd nadat een geslaagde duurzame
schrijfactie de effectieve `nextRunAtMs` van een bestaande taak wijzigt, met uitzondering van de expliciete
levenscyclusgebeurtenis `added`, `updated` of `removed` van die taak. De `event.nextRunAtMs`
op het hoogste niveau is de vastgelegde volgende wekactie; wanneer deze ontbreekt, heeft de taak
geen volgende wekactie. Behandel deze gebeurtenissen als aanwijzingen voor reconciliatie, niet als een geordend deltalogboek.
Gebruik ze als samenvoegbare aanwijzingen om de planner die het laatst door
`cron_reconciled` is vastgelegd opnieuw te lezen; neem de planner niet over uit een `cron_changed`-context.
Behoud OpenClaw als de bron van waarheid voor controles op verschuldigde taken en uitvoering.

### Veilige externe Cron-projectie

Projecteer een volledige momentopname van wekacties in plaats van delta's van Cron-gebeurtenissen door te sturen. De
bewerking `replaceAll` van de externe adapter moet atomair en idempotent zijn en mag
pas worden voltooid nadat de host de momentopname duurzaam heeft geaccepteerd. Deze moet
ook rekening houden met het aangeleverde afbreeksignaal: als het signaal vóór duurzame
acceptatie wordt afgebroken, mag de adapter die momentopname niet accepteren.

Met dit patroon is er één worker voor de nieuwste status actief. Alleen `cron_reconciled`
neemt een plannerinstantie over; `cron_changed` vraagt die worker alleen om
de gezaghebbende instantie opnieuw te lezen, zodat een late aanwijzing geen oudere planner kan herstellen.
Een nieuwere revisie breekt de actieve hostpoging af voordat deze een verouderde
momentopname kan accepteren.

```typescript
import { setTimeout as sleep } from "node:timers/promises";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/plugin-entry";

type ExternalWake = { jobId: string; runAtMs: number };

type ExternalWakeHost = {
  replaceAll(wakes: readonly ExternalWake[], options: { signal: AbortSignal }): Promise<void>;
  close(): Promise<void>;
};

type CronReader = {
  list(options: { includeDisabled: true }): Promise<
    Array<{
      id: string;
      enabled?: boolean;
      state?: { nextRunAtMs?: number };
    }>
  >;
};

export function registerCronProjection(api: OpenClawPluginApi, host: ExternalWakeHost) {
  const lifecycle = new AbortController();
  let cron: CronReader | undefined;
  let enabled = false;
  let hasBaseline = false;
  let reconciliationSignal: AbortSignal | undefined;
  let requestedRevision = 0;
  let appliedRevision = 0;
  let worker = Promise.resolve();
  let activeAttempt: AbortController | undefined;

  const projectLatest = async () => {
    let retryMs = 1_000;

    while (!lifecycle.signal.aborted && appliedRevision < requestedRevision) {
      const ownerSignal = reconciliationSignal;
      if (!ownerSignal || ownerSignal.aborted) {
        return;
      }
      const targetRevision = requestedRevision;
      const attempt = new AbortController();
      const signal = AbortSignal.any([lifecycle.signal, ownerSignal, attempt.signal]);
      activeAttempt = attempt;

      try {
        const jobs = enabled && cron ? await cron.list({ includeDisabled: true }) : [];
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        const wakes = jobs
          .flatMap((job): ExternalWake[] => {
            const runAtMs = job.enabled === false ? undefined : job.state?.nextRunAtMs;
            return runAtMs === undefined ? [] : [{ jobId: job.id, runAtMs }];
          })
          .sort((a, b) => a.runAtMs - b.runAtMs || a.jobId.localeCompare(b.jobId));

        await host.replaceAll(wakes, { signal });
        if (signal.aborted || targetRevision !== requestedRevision) {
          continue;
        }
        appliedRevision = targetRevision;
        retryMs = 1_000;
      } catch {
        if (lifecycle.signal.aborted || ownerSignal.aborted) {
          return;
        }
        if (attempt.signal.aborted) {
          continue;
        }
        api.logger.warn(`externe Cron-projectie mislukt; nieuwe poging over ${retryMs} ms`);
        try {
          await sleep(retryMs, undefined, { signal });
        } catch {
          if (lifecycle.signal.aborted) {
            return;
          }
          if (attempt.signal.aborted) {
            continue;
          }
        }
        retryMs = Math.min(retryMs * 2, 30_000);
      } finally {
        if (activeAttempt === attempt) {
          activeAttempt = undefined;
        }
      }
    }
  };

  const requestProjection = () => {
    const targetRevision = ++requestedRevision;
    activeAttempt?.abort();
    worker = worker.then(async () => {
      if (!lifecycle.signal.aborted && appliedRevision < targetRevision) {
        await projectLatest();
      }
    });
    return worker;
  };

  api.on("cron_reconciled", (event, ctx) => {
    const reconciledCron = ctx.getCron?.();
    if (event.enabled && !reconciledCron) {
      api.logger.warn("Cron-reconciliatie heeft geen planner beschikbaar gesteld");
      return;
    }
    cron = reconciledCron;
    enabled = event.enabled;
    hasBaseline = true;
    reconciliationSignal = ctx.abortSignal;
    return requestProjection();
  });

  api.on("cron_changed", () => {
    if (hasBaseline) {
      return requestProjection();
    }
  });

  api.on("gateway_stop", async () => {
    lifecycle.abort();
    await worker;
    await host.close();
  });
}
```

Wanneer `cron_reconciled` `enabled: false` rapporteert, roept hetzelfde pad
`replaceAll([])` aan en wist het verouderde externe wekacties. Opnieuw proberen/back-off in dit voorbeeld
is proceslokaal en behandelt fouten in runtime-adapters als tijdelijk; valideer
niet-opnieuw-probeerbare configuratie vóór registratie. OpenClaw biedt geen
outbox voor effecten van pluginhooks. Als het proces wordt afgesloten vóór duurzame acceptatie,
zendt de volgende start van de Gateway een nieuwe gezaghebbende `cron_reconciled`-momentopname uit.
`gateway_stop` breekt lopend hostwerk af, wacht tot de worker tot rust is gekomen en
sluit vervolgens de adapter.

## Aankomende afschaffingen

Enkele oppervlakken naast hooks zijn verouderd maar worden nog steeds ondersteund. Migreer
vóór de volgende hoofdversie:

- **Plattetekst-enveloppen voor kanalen** in `inbound_claim`- en `message_received`-
  handlers. Lees `BodyForAgent` en de gestructureerde gebruikerscontextblokken
  in plaats van platte enveloptekst te parseren. Zie
  [Plattetekst-enveloppen voor kanalen → BodyForAgent](/nl/plugins/sdk-migration#active-deprecations).
- **`subagent_spawning`** blijft behouden voor compatibiliteit met oudere plugins, maar
  nieuwe plugins mogen er geen threadroutering uit retourneren. De kern bereidt
  `thread: true`-subagentbindingen voor via adapters voor kanaalsessiebindingen
  voordat `subagent_spawned` wordt geactiveerd.
- **`deactivate`** blijft tot na 2026-08-16 behouden als verouderde compatibiliteitsalias
  voor opschoning. Nieuwe plugins moeten `gateway_stop` gebruiken.
- **`onResolution` in `before_tool_call`** gebruikt nu de getypeerde
  `PluginApprovalResolution`-union (`allow-once` / `allow-always` / `deny` /
  `timeout` / `cancelled`) in plaats van een vrije `string`.
- **`api.registerSessionExtension` / `api.enqueueNextTurnInjection`** blijven
  behouden als compatibiliteitsaliassen op het hoogste niveau. Nieuwe plugins moeten
  `api.session.state.registerSessionExtension(...)` en
  `api.session.workflow.enqueueNextTurnInjection(...)` gebruiken.

Zie voor de volledige lijst — registratie van geheugencapaciteiten, het denkprofiel
van de provider, externe authenticatieproviders, typen voor providerdetectie, accessors
voor de taakruntime en de naamswijziging van `command-auth` → `command-status` —
[Plugin SDK-migratie → Actieve afschrijvingen](/nl/plugins/sdk-migration#active-deprecations).

## Gerelateerd

- [Plugin SDK-migratie](/nl/plugins/sdk-migration) - actieve afschrijvingen en tijdlijn voor verwijdering
- [Plugins bouwen](/nl/plugins/building-plugins)
- [Overzicht van de Plugin SDK](/nl/plugins/sdk-overview)
- [Plugin-toegangspunten](/nl/plugins/sdk-entrypoints)
- [Interne hooks](/nl/automation/hooks)
- [Interne werking van de pluginarchitectuur](/nl/plugins/architecture-internals)
