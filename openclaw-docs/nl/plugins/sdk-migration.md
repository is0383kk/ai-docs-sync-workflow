---
read_when:
    - Je ziet de waarschuwing OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Je ziet de waarschuwing OPENCLAW_EXTENSION_API_DEPRECATED
    - Je gebruikte api.registerEmbeddedExtensionFactory vóór OpenClaw 2026.4.25
    - Je werkt een plugin bij naar de moderne pluginarchitectuur
    - Je onderhoudt een externe OpenClaw-plugin
sidebarTitle: Migrate to SDK
summary: Migreer van de verouderde achterwaartse-compatibiliteitslaag naar de moderne Plugin-SDK
title: Migratie van de Plugin SDK
x-i18n:
    generated_at: "2026-07-27T05:10:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw heeft een brede laag voor achterwaartse compatibiliteit vervangen door een moderne Plugin-
architectuur die is opgebouwd uit kleine, gerichte imports. Als jouw Plugin van vóór die
wijziging dateert, helpt deze handleiding je om over te stappen op de huidige contracten.

## Wat is er gewijzigd

Verschillende zeer ruime importoppervlakken gaven Plugins voorheen vanuit één toegangspunt
toegang tot vrijwel alles:

- **`openclaw/plugin-sdk`** en **`openclaw/plugin-sdk/compat`** - exporteerden
  tientallen helpers opnieuw terwijl de gerichte SDK werd gebouwd. Beide hoofdpaden zijn nu
  verwijderd; importeer in plaats daarvan een gedocumenteerd subpad.
- **`openclaw/plugin-sdk/infra-runtime`** - een brede barrel die systeem-
  gebeurtenissen, Heartbeat-status, afleveringswachtrijen, fetch-/proxyhelpers, bestandshelpers,
  goedkeuringstypen en niet-gerelateerde hulpprogramma's combineerde.
- **`openclaw/plugin-sdk/config-runtime`** - een brede configuratiebarrel die
  alleen behouden bleef voor het latere compatibiliteitsvenster; directe runtimehelpers voor
  laden/schrijven zijn verwijderd.
- **`openclaw/extension-api`** - een verwijderde brug die Plugins directe
  toegang gaf tot helpers aan de hostzijde, zoals de ingebedde agentrunner.
- **`api.registerEmbeddedExtensionFactory(...)`** - een verwijderde hook die uitsluitend voor
  de ingebedde runner was bedoeld en gebeurtenissen daarvan observeerde, zoals `tool_result`. Gebruik in plaats
  daarvan middleware voor agenttoolresultaten (zie [Ingebedde extensies voor toolresultaten
  migreren naar middleware](#how-to-migrate)).

De hoofd-SDK, compatibiliteitsbarrel, extensiebrug en fabriek voor ingebedde extensies
zijn verwijderd. `infra-runtime` en `config-runtime` blijven alleen bestaan voor hun
afzonderlijk vastgelegde latere vensters; nieuwe Plugins moeten gerichte subpaden gebruiken.

<Warning>
  Plugins die de verwijderde hoofd-, compatibiliteits- of extensieoppervlakken importeren, worden niet meer
  geladen. Volg vóór het upgraden de onderstaande omzettingen.
</Warning>

OpenClaw verwijdert of herinterpreteert gedocumenteerd Plugingedrag niet in dezelfde
wijziging waarin een vervanging wordt geïntroduceerd. Incompatibele contractwijzigingen doorlopen
eerst een compatibiliteitsadapter, diagnostiek, documentatie en een uitfaseringsvenster. Dat
geldt voor SDK-imports, manifestvelden, installatie-API's, hooks en runtimegedrag
voor registratie.

### Waarom

- **Langzaam opstarten** - door één helper te importeren, werden tientallen niet-gerelateerde modules geladen.
- **Circulaire afhankelijkheden** - brede herexports maakten het eenvoudig om importcycli
  te creëren.
- **Onduidelijk API-oppervlak** - er was geen manier om stabiele exports van interne exports te onderscheiden.

Elke `openclaw/plugin-sdk/<subpath>` is nu een kleine, zelfstandige module met
een gedocumenteerd contract.

Verouderde gemaksinterfaces voor providers van gebundelde kanalen zijn ook verdwenen -
helpersnelkoppelingen met kanaalnamen waren private gemakken voor de monorepo, geen
stabiele Plugincontracten. Gebruik in plaats daarvan smalle, algemene SDK-subpaden. Houd binnen de
werkruimte van de gebundelde Plugin helpers die eigendom zijn van de provider in de eigen
`api.ts` of `runtime-api.ts` van die Plugin:

- Anthropic bewaart Claude-specifieke streamhelpers in zijn eigen `api.ts` /
  `contract-api.ts`-interface.
- OpenAI bewaart providerbouwers, helpers voor standaardmodellen en realtime
  providerbouwers in zijn eigen `api.ts`.
- OpenRouter bewaart de providerbouwer en helpers voor onboarding/configuratie in zijn eigen
  `api.ts`.

## Compatibiliteitsbeleid

Compatibiliteitswerk voor externe Plugins volgt deze volgorde:

1. Voeg het nieuwe contract toe.
2. Behoud het oude gedrag via een compatibiliteitsadapter.
3. Geef een diagnostisch bericht of waarschuwing weer waarin het oude pad en de vervanging worden genoemd.
4. Dek beide paden af met tests.
5. Documenteer de uitfasering en het migratiepad.
6. Verwijder pas na het aangekondigde migratievenster, meestal in een hoofd-
   release.

Als een manifestveld nog wordt geaccepteerd, blijf het dan gebruiken totdat de documentatie en
diagnostiek anders aangeven. Nieuwe code moet de gedocumenteerde vervanging gebruiken;
bestaande Plugins mogen niet defect raken tijdens normale kleinere releases.

### Compatibiliteit voor installatie van gepubliceerde kanalen

Slack-, Discord-, Signal- en Microsoft Teams-pakketten die via
`2026.7.1` worden gepubliceerd, importeren kanaalspecifieke configuratieschema's uit
`openclaw/plugin-sdk/bundled-channel-config-schema`. De gepubliceerde Slack- en
Discord-pakketten importeren ook `createLegacyCompatChannelDmPolicy` en
`promptLegacyChannelAllowFromForAccount` uit
`openclaw/plugin-sdk/setup-runtime`.

Die exports blijven beschikbaar als verouderde runtimecompatibiliteitsadapters.
Nieuwe en opnieuw gepubliceerde Plugins moeten hun configuratieschema's en installatiebeleid
lokaal beheren, met algemene primitieve onderdelen uit `channel-config-schema` en
`setup-runtime`. De compatibiliteitsexports kunnen pas worden verwijderd wanneer de
minimaal ondersteunde versies van gepubliceerde pakketten ze niet meer importeren.

### Compatibiliteit van invoervelden voor kanaalinstallatie

`ChannelSetupInput` houdt nu alleen de kanaaloverstijgende installatie-envelop permanent
getypeerd. Kanaalspecifieke velden blijven getypeerd in een verouderde compatibiliteits-
laag, zodat bestaande externe Plugins blijven compileren terwijl Pluginauteurs die
velden verplaatsen naar lokale invoertypen voor de installatie van hun Plugin.

OpenClaw brengt geen hoofdreleases uit. Bij een registersweep op 2026-07-22 werden
426 gepubliceerde externe kanaal-Plugins geïnspecteerd en 21 velden zonder lezers verwijderd.
De 22 behouden velden hebben elk een bekende gepubliceerde lezer. Elk volgend veld wordt
verwijderd zodra geen gepubliceerde Plugin het meer leest; de behouden verzameling krimpt
naarmate Pluginauteurs migreren naar lokale invoertypen voor Plugininstallatie.

Dezelfde sweep verwijderde 23 verouderde promotiesleutels voor niet-gedeclareerde adapters zonder
gepubliceerde afhankelijken. Zes algemene sleutels en de uitsluitend voor installatie bedoelde sleutel `rooms` blijven bestaan.
Ook die verzameling krimpt naarmate gepubliceerde Plugins `singleAccountKeysToMove` declareren.

Het gedeelde type heeft geen indexsignatuur. Sleutels die eigendom zijn van Plugins kunnen nog steeds aanwezig zijn
in runtime-invoerobjecten; declareer ze in een lokale intersectie voor de Plugin of beperk
ze via het installatieschema van de eigenaar-Plugin.

| `code`                                  | `owner`   | `replacement`                                                                                    | Voorwaarde voor verwijdering                                           |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | Combineer `ChannelSetupInput` met een lokaal type voor de Plugin dat de velden van het betreffende kanaal declareert | Verwijder een veld wanneer de registersweep van gepubliceerde Plugins geen lezer vindt |

De verouderde promotielaag voor niet-gedeclareerde adapters volgt hetzelfde
beleid op basis van lezers. Declareer `singleAccountKeysToMove`, inclusief een lege array wanneer de
Plugin geen extra promotiesleutels nodig heeft, zodat de gedeelde fallback sleutel voor sleutel kan worden uitgefaseerd.

#### Lezers verifiëren

1. Doorloop `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` pagina voor pagina met elke `nextCursor` en behoud pakketten waarvan de `categories` `channels` bevatten.
2. Voeg npm-kandidaten uit `npm search --json --searchlimit=1000 "openclaw channel plugin"` toe. Voeg kandidaten toe die alleen als bron beschikbaar zijn via GitHub-codezoekopdrachten naar `openclaw/plugin-sdk/channel-setup`, `openclaw/plugin-sdk/setup` en `openclaw/plugin-sdk/core`.
3. Bepaal voor elke kandidaat de laatst gepubliceerde versie. Voer `npm pack <package>@<version> --json --pack-destination <temp-dir>` uit, pak het pakket uit en inspecteer de meegeleverde `dist`-JavaScript en declaraties op directe of gedestructureerde leesbewerkingen van velden. Download het ClawHub-artefact wanneer een pakket geen npm-release heeft.
4. Leg pakket, versie, veld of promotiesleutel en overeenkomend bestand vast. Een veld of sleutel kan alleen worden verwijderd wanneer geen gepubliceerd Pluginartefact het leest. Houd de namen van lezers in de codeopmerkingen naast de lijsten met behouden velden en sleutels gesynchroniseerd met de sweep.

Dit is uitsluitend een compatibiliteitsregistratie voor broncode/typen. Er is geen runtimeadapter of
vermelding in het compatibiliteitsregister, omdat runtime-invoerobjecten voor installatie en het installatiegedrag
ongewijzigd zijn.

Controleer de huidige migratiewachtrij met `pnpm plugins:boundary-report`:

| Vlag                                                    | Effect                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (of `pnpm plugins:boundary-report:summary`) | Compacte aantallen in plaats van volledige details.                            |
| `--json`                                                | Machineleesbaar rapport.                                                       |
| `--owner <id>`                                          | Filter op één Plugin of compatibiliteitseigenaar.                              |
| `--fail-on-cross-owner`                                 | Sluit af met een niet-nulstatus bij gereserveerde SDK-imports tussen eigenaren. |
| `--fail-on-eligible-compat`                             | Sluit af met een niet-nulstatus wanneer de datum `removeAfter` van een verouderde compatibiliteitsregistratie is verstreken. |
| `--fail-on-unclassified-unused-reserved`                | Sluit af met een niet-nulstatus bij ongebruikte gereserveerde SDK-shims.       |

`pnpm plugins:boundary-report:ci` wordt uitgevoerd met alle drie foutvlaggen. Verouderde
registraties hebben normaal gesproken een expliciete datum `removeAfter` in plaats van een vage 'volgende
hoofdrelease'. Bij een registratie waarvan de eigenaar geen datum heeft goedgekeurd, ontbreekt
`removeAfter`, wordt `no-date` weergegeven en is verwijdering nooit toegestaan.
Het rapport groepeert verouderde registraties op datum, telt lokale verwijzingen in code/documentatie,
toont gereserveerde SDK-imports tussen eigenaren en vat de private
SDK-brug voor de geheugenhost samen. Gereserveerde SDK-subpaden moeten bijgehouden gebruik door eigenaren hebben;
ongebruikte gereserveerde exports moeten uit de openbare SDK worden verwijderd.

### Verouderde mediaprojectie

De compatibiliteitsregistratie `media-legacy-projection` omvat de oude parallelle
mediavelden, payloadbouwers, aliassen voor hookmetadata en namen van mediasjablonen.
De goedgekeurde datum `removeAfter` is **2026-10-01** (twee releasetreinen
nadat de op feiten gebaseerde vervangingen zijn uitgebracht). Voor verwijdering is daarnaast
op dat moment een schone sweep van gepubliceerde Pluginartefacten vereist; migreer vóór die datum.

Vervang voor binnenkomende kanaalgegevens de enkelvoudige/meervoudige `MediaPath`, `MediaUrl`,
`MediaType`, `MediaPaths`, `MediaUrls`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` en `MediaStaged` door geordende
feiten:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

Gebruik `event.media` in de hooks `inbound_claim` en `message_received`. Als externe
media niet lokaal zijn klaargezet, gebruik dan `event.originalMedia` voor identiteit/diagnostiek
en wacht op `event.media`; `event.mediaStagingPending` onderscheidt die
status. Lees de verouderde enkelvoudige/meervoudige eigenschappen niet uit
`event.metadata`.

Vervang voor CLI-mediamodellen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`
en `{{MediaDir}}` door `{{AttachmentPath}}`, `{{AttachmentUrl}}`,
`{{AttachmentContentType}}` en `{{AttachmentDir}}`. Gebruik
`{{AttachmentIndex}}` wanneer de positie van de bijlage van belang is.

Importeer voor lokaal beleid voor het lezen van media `getAgentScopedMediaLocalRoots(...)` of
`getAgentScopedMediaLocalRootsForSources(...)` uit
`openclaw/plugin-sdk/media-local-roots`. De
`openclaw/plugin-sdk/agent-media-payload`-facade en de bijbehorende
`buildAgentMediaPayload(...)`-projectie zijn verouderd.

## Migreren

<Steps>
  <Step title="Runtimehelpers voor laden/schrijven van configuratie migreren">
    Gebundelde Plugins mogen `api.runtime.config.loadConfig()` en
    `api.runtime.config.writeConfigFile(...)` niet meer rechtstreeks aanroepen. Gebruik bij voorkeur de configuratie die al
    aan het actieve aanroeppad is doorgegeven. Langlevende handlers die de
    huidige processnapshot nodig hebben, kunnen `api.runtime.config.current()` gebruiken. Langlevende
    agenttools moeten `ctx.getRuntimeConfig()` binnen `execute` lezen, zodat een tool
    die vóór een configuratieschrijfactie is gemaakt, toch de vernieuwde configuratie ziet.

    Configuratieschrijfacties verlopen via de transactionele helper met een expliciet
    beleid voor na het schrijven:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    Gebruik `afterWrite: { mode: "restart", reason: "..." }` wanneer de wijziging
    een schone herstart van de Gateway vereist, en `afterWrite: { mode: "none", reason: "..." }`
    alleen wanneer de aanroeper verantwoordelijk is voor de vervolgactie en de
    herlaadplanner bewust onderdrukt. Mutatieresultaten bevatten een getypeerd `followUp`-overzicht voor
    tests en logboekregistratie; de Gateway blijft verantwoordelijk voor het toepassen of
    plannen van de herstart.

    `loadConfig` en `writeConfigFile` zijn verwijderd uit de Plugin-
    runtime. Gebundelde plugins en runtimecode van de repository worden bewaakt door
    `pnpm check:deprecated-api-usage` en
    `pnpm check:no-runtime-action-load-config`: nieuw productiegebruik door plugins
    mislukt direct, rechtstreekse configuratieschrijfbewerkingen mislukken, Gateway-servermethoden moeten
    de runtime-snapshot van het verzoek gebruiken, runtimehelpers voor verzenden/acties/clients van kanalen
    moeten configuratie vanuit hun grens ontvangen en langlevende runtimemodules
    staan nul omgevingsaanroepen naar `loadConfig()` toe.

    Nieuwe plugincode moet de brede `openclaw/plugin-sdk/config-runtime`-
    barrel vermijden. Gebruik het specifieke subpad voor de taak:

    | Behoefte | Import |
    | --- | --- |
    | Configuratietypen zoals `OpenClawConfig` | `openclaw/plugin-sdk/config-contracts` |
    | Configuratieopzoeking bij Plugin-invoer | `api.pluginConfig` |
    | Configuratie samenvoegen | Plugin-lokale logica bij de configuratiegrens |
    | Huidige runtime-snapshot lezen | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | Configuratie schrijven | `openclaw/plugin-sdk/config-mutation` |
    | Helpers voor sessieopslag | `openclaw/plugin-sdk/session-store-runtime` |
    | Configuratie voor Markdown-tabellen | `openclaw/plugin-sdk/markdown-table-runtime` |
    | Runtimehelpers voor groepsbeleid | `openclaw/plugin-sdk/runtime-group-policy` |
    | Geheime invoer oplossen | `openclaw/plugin-sdk/secret-input-runtime` |
    | Model-/sessieoverschrijvingen | `openclaw/plugin-sdk/model-session-runtime` |

    Gebundelde plugins en hun tests worden door scanners beschermd tegen de brede
    barrel, zodat imports en mocks lokaal blijven voor het gedrag dat ze nodig hebben. De
    barrel bestaat nog steeds voor externe compatibiliteit, maar nieuwe code mag er niet
    van afhankelijk zijn.

  </Step>

  <Step title="Ingebedde extensies voor toolresultaten naar middleware migreren">
    Gebundelde plugins moeten de uitsluitend voor ingebedde runners bedoelde
    `api.registerEmbeddedExtensionFactory(...)`-handlers voor toolresultaten vervangen door
    runtime-neutrale middleware:

    ```typescript
    // OpenClaw-runtimetools en dynamische tools van de Codex-runtime (resultaat kan worden
    // getransformeerd). Codex-native toolresultaten worden ook doorgestuurd voor observatie,
    // maar hun getransformeerde uitvoer bereikt het model nooit: het contract van de Codex-
    // PostToolUse-hook kan een native toolantwoord niet vervangen.
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    Werk tegelijkertijd het Plugin-manifest bij:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    Geïnstalleerde plugins kunnen ook middleware voor toolresultaten registreren wanneer dit expliciet
    is ingeschakeld en elke beoogde runtime is gedeclareerd in
    `contracts.agentToolResultMiddleware`. Niet-gedeclareerde middleware-
    registraties van geïnstalleerde plugins worden geweigerd.

  </Step>

  <Step title="Native goedkeuringshandlers naar capaciteitsfeiten migreren">
    Kanaalplugins met ondersteuning voor goedkeuring stellen native goedkeuringsgedrag beschikbaar via
    `approvalCapability.nativeRuntime` plus het gedeelde register voor
    runtimecontext:

    - Vervang `approvalCapability.handler.loadRuntime(...)` door
      `approvalCapability.nativeRuntime`.
    - Verplaats goedkeuringsspecifieke authenticatie/bezorging van de verouderde `plugin.auth`-/
      `plugin.approvals`-bedrading naar `approvalCapability`.
    - `ChannelPlugin.approvals` is verwijderd uit het openbare
      kanaalplugincontract; verplaats bezorgings-, native en renderingsvelden naar
      `approvalCapability`.
    - `plugin.auth` blijft alleen voor aanmeld-/afmeldflows van kanalen; de kern
      leest daar geen authenticatiehooks voor goedkeuring meer.
    - Registreer runtimeobjecten die eigendom zijn van het kanaal (clients, tokens, Bolt-apps)
      via `openclaw/plugin-sdk/channel-runtime-context`.
    - Verzend geen omleidingsmeldingen die eigendom zijn van de Plugin vanuit native goedkeuringshandlers;
      de kern beheert meldingen over routering naar elders op basis van werkelijke bezorgingsresultaten.
    - Wanneer je `channelRuntime` doorgeeft aan `createChannelManager(...)`, geef dan een
      echte `createPluginRuntime().channel`-interface op; gedeeltelijke stubs worden
      geweigerd.

    Zie [Kanaalplugins](/nl/plugins/sdk-channel-plugins) voor de huidige
    indeling van goedkeuringscapaciteiten.

  </Step>

  <Step title="Fallbackgedrag van Windows-wrappers controleren">
    Als je Plugin `openclaw/plugin-sdk/windows-spawn` gebruikt, mislukken niet-opgeloste Windows-
    wrappers voor `.cmd`/`.bat` nu standaard veilig, tenzij je expliciet
    `allowShellFallback: true` doorgeeft:

    ```typescript
    // Voorheen
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Nu
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Stel dit alleen in voor vertrouwde compatibiliteitsaanroepers die bewust
      // een door de shell bemiddelde fallback accepteren.
      allowShellFallback: true,
    });
    ```

    Als je aanroeper niet bewust afhankelijk is van een shellfallback, stel
    `allowShellFallback` dan niet in en handel in plaats daarvan de gegenereerde fout af.

  </Step>

  <Step title="Verouderde imports zoeken">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="Vervangen door gerichte imports">
    Elke export van de oude interface correspondeert met een specifiek modern importpad:

    ```typescript
    // Voorheen (verouderde laag voor achterwaartse compatibiliteit)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Nu (moderne gerichte imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Gebruik voor helpers aan de hostzijde de geïnjecteerde Plugin-runtime in plaats van
    rechtstreeks te importeren:

    ```typescript
    // Voorheen (verouderde extension-api-brug)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // Nu (geïnjecteerde runtime)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    Hetzelfde patroon geldt voor andere verouderde brughelpers:

    | Oude import | Modern equivalent |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | helpers voor sessieopslag | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Brede infra-runtime-imports vervangen">
    `openclaw/plugin-sdk/infra-runtime` bestaat nog steeds voor externe
    compatibiliteit, maar nieuwe code moet de gerichte interface importeren die daadwerkelijk
    nodig is:

    | Behoefte | Import |
    | --- | --- |
    | Helpers voor de wachtrij met systeemgebeurtenissen | `openclaw/plugin-sdk/system-event-runtime` |
    | Helpers voor Heartbeat-activering, gebeurtenissen en zichtbaarheid | `openclaw/plugin-sdk/heartbeat-runtime` |
    | Wachtrij met wachtende bezorgingen leegmaken | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | Telemetrie voor kanaalactiviteit | `openclaw/plugin-sdk/channel-activity-runtime` |
    | Dedupe-caches in het geheugen en met persistente opslag | `openclaw/plugin-sdk/dedupe-runtime` |
    | Veilige helpers voor lokale bestands-/mediapaden | `openclaw/plugin-sdk/file-access-runtime` |
    | Dispatcherbewuste fetch | `openclaw/plugin-sdk/runtime-fetch` |
    | Helpers voor proxy-fetch en beveiligde fetch | `openclaw/plugin-sdk/fetch-runtime` |
    | Beleidstypen voor SSRF-dispatchers | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | Typen voor goedkeuringsverzoeken/-afhandelingen | `openclaw/plugin-sdk/approval-runtime` |
    | Helpers voor antwoordpayloads en opdrachten bij goedkeuring | `openclaw/plugin-sdk/approval-reply-runtime` |
    | Helpers voor foutopmaak | `openclaw/plugin-sdk/error-runtime` |
    | Wachten op transportgereedheid | `openclaw/plugin-sdk/transport-ready-runtime` |
    | Helpers voor veilige tokens | `openclaw/plugin-sdk/secure-random-runtime` |
    | Begrensde gelijktijdigheid van asynchrone taken | `openclaw/plugin-sdk/concurrency-runtime` |
    | Vereiste-waardecontroles voor bewijsbare invarianten | `openclaw/plugin-sdk/expect-runtime` |
    | Numerieke conversie | `openclaw/plugin-sdk/number-runtime` |
    | Proceslokale asynchrone vergrendeling | `openclaw/plugin-sdk/async-lock-runtime` |
    | Bestandsvergrendelingen | `openclaw/plugin-sdk/file-lock` |

    Gebundelde plugins worden door scanners beschermd tegen `infra-runtime`, zodat repositorycode
    niet kan terugvallen op de brede barrel.

  </Step>

  <Step title="Helpers voor kanaalroutes migreren">
    Nieuwe code voor kanaalroutes gebruikt `openclaw/plugin-sdk/channel-route`. De oudere
    route-sleutelnamen blijven beschikbaar als compatibiliteitsaliassen:

    | Oude helper | Moderne helper |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    De moderne routehelpers normaliseren `{ channel, to, accountId, threadId }`
    consistent voor native goedkeuringen, onderdrukking van antwoorden, deduplicatie van inkomende berichten,
    Cron-bezorging en sessieroutering.

    Voeg geen nieuw gebruik toe van `ChannelMessagingAdapter.parseExplicitTarget` of
    `resolveChannelRouteTargetWithParser(...)` uit
    `plugin-sdk/channel-route`; deze zijn verouderd en blijven alleen voor oudere
    plugins bestaan. Nieuwe kanaalplugins moeten
    `messaging.targetResolver.resolveTarget(...)` gebruiken voor normalisatie van doel-id's
    en fallback bij ontbrekende directoryvermeldingen,
    `messaging.inferTargetChatType(...)` wanneer de kern vroegtijdig een peertype nodig heeft,
    en `messaging.resolveOutboundSessionRoute(...)` voor provider-native
    sessie- en threadidentiteit.

  </Step>

  <Step title="Bouwen en testen">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## Naslag voor importpaden

De exportmap van het openbare pakket is de gezaghebbende bron voor importeerbare SDK-
subpaden. Gebruik de thematische SDK-handleidingen waarnaar wordt verwezen vanuit het [SDK-overzicht](/nl/plugins/sdk-overview)
en geef de voorkeur aan het specifiekste gedocumenteerde openbare subpad. De compilerinventaris in
`scripts/lib/plugin-sdk-entrypoints.json` bevat ook privé-lokale vermeldingen die worden gebruikt
om gebundelde plugins te bouwen; hun aanwezigheid daarin maakt ze niet tot openbare pakketexports.

Deze tabel is de gebruikelijke migratiesubset, niet de volledige SDK-interface. De
inventaris met compilerinvoerpunten staat in `scripts/lib/plugin-sdk-entrypoints.json`;
pakketexports worden gegenereerd uit de openbare subset.

Gereserveerde helperinterfaces voor gebundelde plugins zijn uit de openbare SDK-
exportmap verwijderd, met uitzondering van expliciet gedocumenteerde compatibiliteitsfacades zoals de
verouderde `plugin-sdk/discord`-shim die behouden blijft voor externe plugins die nog steeds
rechtstreeks het gepubliceerde pakket `@openclaw/discord` importeren. Eigenaarsspecifieke
helpers bevinden zich in het pakket van de eigenaar; gedeeld hostgedrag verloopt
via generieke SDK-contracten zoals `plugin-sdk/gateway-runtime`,
`plugin-sdk/security-runtime` en de geïnjecteerde Plugin-API.

Gebruik de specifiekste import die bij de taak past. Als je geen export kunt vinden,
controleer dan de bron in `src/plugin-sdk/` of vraag de beheerders welk generiek
contract er eigenaar van moet zijn.

## Verwijderde compatibiliteitsinterfaces

Tijdens de opschoningsronde van juli 2026 zijn de root-SDK- en compat-barrels, de extension API-
brug, de verlopen aliassen voor SDK-subpaden, ongebruikte SDK-subpaden en de openbare
exports voor uitsluitend gebundelde SDK-modules verwijderd. Uitsluitend gebundelde modules blijven voor
hun repository-eigenaren beschikbaar via privé-lokale bouwtoewijzingen; ze zijn niet
importeerbaar vanuit het gepubliceerde pakket.

### Procesglobale publicatie van API-providers

`registerApiProvider(...)` en `unregisterApiProviders(...)` zijn verwijderd uit
`openclaw/plugin-sdk/llm`. Ze publiceerden API-transporten in procesglobale
status, die modelruntimes met een eigen levenscyclus vervolgens naar elk voorbereid
register moesten kopiëren.

Providerplugins moeten providers voor tekstinferentie registreren via
`api.registerProvider(...)`. Code en tests die eigendom zijn van de host en een
`ApiRegistry` aanmaken, moeten rechtstreeks in dat register registreren, zodat providereigendom
en afbouw beperkt blijven tot de voorbereide runtime.

### Privétestbarrel

`openclaw/plugin-sdk/testing` was repository-lokaal en uitgesloten van uitgebrachte pakket-
artefacten, en is daarom vóór de `removeAfter`-datum van 2026-07-28 verwijderd. Repository-
tests gebruiken gerichte subpaden zoals `plugin-sdk/plugin-test-runtime`,
`plugin-sdk/channel-test-helpers`, `plugin-sdk/channel-target-testing`,
`plugin-sdk/test-env` en `plugin-sdk/test-fixtures`.

## Migratienaslag

  Deze toewijzingen omvatten zowel in juli 2026 verwijderde oppervlakken als actieve afschrijvingen met een latere termijn. Een toewijzing is migratierichtlijn, geen bewijs dat het oude oppervlak beschikbaar blijft; raadpleeg het compatibiliteitsregister en de verwijderingstijdlijn voor de huidige status.

  <AccordionGroup>
  <Accordion title="Helpers voor command-auth-help -> command-status">
    **Oud (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`.

    **Nieuw (`openclaw/plugin-sdk/command-status`)**: dezelfde signaturen, geïmporteerd
    vanuit het specifiekere subpad. De compatibiliteitsherexports van
    `command-auth` zijn verwijderd.

    ```typescript
    // Voorheen
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // Nu
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="Helpers voor vermeldingscontrole -> resolveInboundMentionDecision">
    **Oud**: `resolveMentionGating(params)` en
    `resolveMentionGatingWithBypass(params)` uit
    `openclaw/plugin-sdk/channel-inbound` of
    `openclaw/plugin-sdk/channel-mention-gating`.

    **Nieuw**: `resolveInboundMentionDecision({ facts, policy })` — één beslissingsobject
    in plaats van twee afzonderlijke aanroepvormen.

    In gebruik genomen voor Discord, iMessage, Matrix, MS Teams, QQBot, Signal,
    Telegram, WhatsApp en Zalo. Slacks eigen `app_mention`-gebeurtenismodel
    gebruikt deze helper niet.

  </Accordion>

  <Accordion title="Shim voor de kanaalruntime en helpers voor kanaalacties">
    `openclaw/plugin-sdk/channel-runtime` is verwijderd. Gebruik
    `openclaw/plugin-sdk/channel-runtime-context` om runtimeobjecten te registreren.

    De helpers voor het systeemeigen berichtschema in `openclaw/plugin-sdk/channel-actions`
    zijn samen met de onbewerkte kanaalexports voor "actions" verwijderd. Stel
    mogelijkheden in plaats daarvan beschikbaar via het semantische
    `presentation`-oppervlak — kanaalplugins geven aan wat ze weergeven
    (kaarten, knoppen, selecties) in plaats van welke onbewerkte actienamen ze
    accepteren.

  </Accordion>

  <Accordion title="Webzoekproviderhelper tool() -> createTool() op de plugin">
    **Oud**: `tool()`-factory uit `openclaw/plugin-sdk/provider-web-search`.

    **Nieuw**: implementeer `createTool(...)` rechtstreeks op de providerplugin.
    OpenClaw heeft de SDK-helper niet langer nodig om de toolwrapper te registreren.

  </Accordion>

  <Accordion title="Plattetekst-enveloppen voor kanalen -> BodyForAgent">
    **Oud**: `api.runtime.channel.reply.formatInboundEnvelope(...)` (en het
    `channelEnvelope`-veld op inkomende berichtobjecten) om van inkomende
    kanaalberichten een platte plattetekst-promptenvelop te maken.

    **Nieuw**: `BodyForAgent` plus gestructureerde gebruikerscontextblokken.
    Kanaalplugins voegen routeringsmetadata (thread, onderwerp, antwoord-op,
    reacties) toe als getypeerde velden in plaats van ze samen te voegen tot een
    prompttekenreeks. De helper `formatAgentEnvelope(...)` wordt nog steeds ondersteund
    voor gegenereerde, op de assistent gerichte enveloppen, maar inkomende
    plattetekst-enveloppen worden uitgefaseerd.

    Betrokken gebieden: `inbound_claim`, `message_received` en elke aangepaste
    kanaalplugin die de oude enveloptekst nabewerkte.

  </Accordion>

  <Accordion title="deactivate-hook -> gateway_stop">
    **Oud**: `api.on("deactivate", handler)`.

    **Nieuw**: `api.on("gateway_stop", handler)`. Hetzelfde contract voor opschoning bij
    afsluiten; alleen de hooknaam verandert.

    ```typescript
    // Voorheen
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // Nu
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` blijft als afgeschreven compatibiliteitsalias gekoppeld
    totdat deze na 2026-08-16 wordt verwijderd.

  </Accordion>

  <Accordion title="subagent_spawning-hook -> kernbinding van threads">
    **Oud**: `api.on("subagent_spawning", handler)` retourneert
    `threadBindingReady` of `deliveryOrigin`.

    **Nieuw**: laat de kern `thread: true`-subagentbindingen voorbereiden via
    de adapter voor kanaalsessiebindingen. Gebruik `api.on("subagent_spawned", handler)` alleen voor
    observatie na het starten.

    ```typescript
    // Voorheen
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // Nu
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`, `PluginHookSubagentSpawningEvent`,
    `PluginHookSubagentSpawningResult` en
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` blijven alleen bestaan als
    afgeschreven compatibiliteitsoppervlakken terwijl externe plugins migreren,
    en worden na 2026-08-30 verwijderd.

  </Accordion>

  <Accordion title="Typen voor providerdetectie -> typen voor providercatalogi">
    Vier typealiassen voor detectie zijn nu dunne wrappers rond de typen uit het
    catalogustijdperk:

    | Oude alias                | Nieuw type                 |
    | ------------------------- | -------------------------- |
    | `ProviderDiscoveryOrder`        | `ProviderCatalogOrder`         |
    | `ProviderDiscoveryContext`        | `ProviderCatalogContext`         |
    | `ProviderDiscoveryResult`        | `ProviderCatalogResult`         |
    | `ProviderPluginDiscovery`        | `ProviderPluginCatalog`         |

    De aliassen en de verouderde statische verzameling `ProviderCapabilities` zijn
    verwijderd. Providerplugins moeten expliciete providerhooks gebruiken, zoals
    `buildReplayPolicy`, `normalizeToolSchemas` en `wrapStreamFn`, in plaats van
    een statisch object.

  </Accordion>

  <Accordion title="Hooks voor denkbeleid -> resolveThinkingProfile">
    **Oud** (drie afzonderlijke hooks op `ProviderThinkingPolicy`):
    `isBinaryThinking(ctx)`, `supportsXHighThinking(ctx)` en
    `resolveDefaultThinkingLevel(ctx)`.

    **Nieuw**: één `resolveThinkingProfile(ctx)` die een
    `ProviderThinkingProfile` retourneert met de canonieke `id`, een
    optionele `label` en een gerangschikte niveaulijst. OpenClaw
    verlaagt verouderde opgeslagen waarden automatisch op basis van de
    profielrang.

    De context bevat `provider`, `modelId`, optionele
    samengevoegde `reasoning`-feiten en optionele samengevoegde
    `compat`-feiten over het model. Providerplugins kunnen die
    catalogusfeiten gebruiken om alleen een modelspecifiek profiel beschikbaar
    te stellen wanneer het geconfigureerde aanvraagcontract dit ondersteunt.

    Implementeer één hook in plaats van drie. De verouderde hooks zijn verwijderd.

  </Accordion>

  <Accordion title="Externe authenticatieproviders -> contracts.externalAuthProviders">
    **Oud**: externe authenticatiehooks implementeren zonder de provider in het
    pluginmanifest te declareren.

    **Nieuw**: declareer `contracts.externalAuthProviders` in het pluginmanifest
    **en** implementeer `resolveExternalAuthProfiles(...)`.

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="Opzoeken van provideromgevingsvariabelen -> setup.providers[].envVars">
    **Oud** manifestveld: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`.

    **Nieuw**: spiegel dezelfde opzoekactie voor omgevingsvariabelen naar
    `setup.providers[].envVars` in het manifest. Dit brengt omgevingsmetadata voor
    installatie en status op één plaats samen en voorkomt dat de pluginruntime
    alleen voor het opzoeken van omgevingsvariabelen moet worden gestart.

    `providerAuthEnvVars` wordt niet langer geaccepteerd.

  </Accordion>

  <Accordion title="Registratie van geheugenplugins -> registerMemoryCapability">
    **Oud**: drie afzonderlijke aanroepen — `api.registerMemoryPromptSection(...)`,
    `api.registerMemoryFlushPlan(...)`, `api.registerMemoryRuntime(...)`.

    **Nieuw**: één aanroep op de API voor geheugenstatus —
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`.

    Dezelfde posities, één registratieaanroep. Aanvullende helpers voor prompts
    en corpora (`registerMemoryPromptSupplement`, `registerMemoryCorpusSupplement`) worden niet beïnvloed.

  </Accordion>

  <Accordion title="API voor providers van geheugeninsluitingen">
    **Oud**: `api.registerMemoryEmbeddingProvider(...)` plus
    `contracts.memoryEmbeddingProviders`.

    **Nieuw**: `api.registerEmbeddingProvider(...)` plus
    `contracts.embeddingProviders`.

    Het generieke contract voor providers van insluitingen is buiten het
    geheugen herbruikbaar en vormt het ondersteunde pad voor nieuwe providers.
    De geheugenspecifieke registratie-API blijft als afgeschreven
    compatibiliteitslaag gekoppeld terwijl bestaande providers migreren.
    Plugininspectie rapporteert niet-gebundeld gebruik als compatibiliteitsschuld.

  </Accordion>

  <Accordion title="Onbewerkte kanaalverzendresultaten -> OutboundDeliveryResult">
    **Oud**: retourneer `{ ok, messageId, error }` via
    `ChannelSendRawResult` en normaliseer het met
    `createRawChannelSendResultAdapter(...)`.

    **Nieuw**: retourneer `OutboundDeliveryResult`-velden en voeg het kanaal toe met
    `createAttachedChannelResultAdapter(...)`. Mislukte verzendingen moeten een uitzondering genereren
    in plaats van een fouttekenreeks te retourneren. Het onbewerkte resultaattype
    blijft beschikbaar tot de volgende hoofdversie van de plugin-SDK.

  </Accordion>

  <Accordion title="Typen voor subagentsessieberichten hernoemd">
    Twee verouderde typealiassen worden nog steeds geëxporteerd vanuit
    `src/plugins/runtime/types.ts`:

    | Oud                           | Nieuw                           |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`            | `SubagentGetSessionMessagesParams`              |
    | `SubagentReadSessionResult`            | `SubagentGetSessionMessagesResult`              |

    De runtimemethode `readSession` is afgeschreven ten gunste van
    `getSessionMessages`. Dezelfde signatuur; de oude methode roept de nieuwe aan.

  </Accordion>

  <Accordion title="Verwijderde API's voor sessie- en transcriptbestanden">
    De omschakeling van sessies en transcripten naar SQLite verwijdert of
    schrijft plugingerichte API's af die actieve `sessions.json`-opslagen,
    paden naar JSONL-transcripten of lijsten met sessiebestanden beschikbaar
    stelden. Runtimeplugins moeten sessie-identiteit en SDK-runtimehelpers
    gebruiken in plaats van actieve bestanden op te zoeken of te wijzigen.

    | Te migreren oppervlak | Vervanging |
    | --------------------- | ---------- |
    | Afgeschreven `loadSessionStore(...)`, `updateSessionStore(...)` en `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`, `listSessionEntries(...)` en sessiewijzigingen op rijniveau. |
    | Afgeschreven `resolveSessionFilePath(...)` | Sessie-identiteit (`sessionKey`, `sessionId` en SDK-runtimehelpers voor doelen) plus Gateway-methoden die op de huidige sessie werken. |
    | Verwijderd `saveSessionStore(...)` | Door de Gateway beheerde sessieruntime-API's; plugincode moet sessiestatus aanvragen of wijzigen via gedocumenteerde runtime-/contexthelpers in plaats van naar het actieve opslagbestand te schrijven. |
    | Verwijderd `resolveSessionTranscriptPathInDir(...)` en `resolveAndPersistSessionFile(...)` | Sessie-identiteit en Gateway-methoden die op de huidige sessie werken. |
    | `readLatestAssistantTextFromSessionTranscript(...)` | Op identiteit gebaseerde transcriptlezers die door de huidige runtimecontext beschikbaar worden gesteld, of Gateway-methoden voor geschiedenis/sessies wanneer de plugin zich buiten het eigenaarspad van het transcript bevindt. |
    | `SessionTranscriptUpdate.sessionFile` | `SessionTranscriptUpdate.target` met `agentId`, `sessionKey` en `sessionId`. |
    | Invoer voor geheugensynchronisatie, zoals `sessionFiles` | Door de host geleverde, op identiteit gebaseerde transcript-/sessiebronnen; doorzoek geen actieve JSONL-bestanden voor live sessies. |
    | Runtimeopties met de naam `transcriptPath` of `sessionFile` voor actieve sessies | `sessionTarget`/runtimedoelobjecten die opslagneutrale sessie-identiteit bevatten. |

    Verouderde JSONL-transcriptbestanden blijven geldig als import-, archief-,
    export- en ondersteuningsartefacten. Ze vormen niet langer het permanente
    runtimecontract voor actieve sessies.

    Officiële plugins die met `v2026.7.1-beta.5` zijn uitgebracht, importeerden
    de vier bovenstaande afgeschreven helpers. `openclaw/plugin-sdk/session-store-runtime` behoudt
    precies die brug tot en met 2026-10-12; nieuwe plugins moeten de vervangingen
    gebruiken. `resolveStorePath(...)` blijft een ondersteunde SDK-helper en maakt
    geen deel uit van deze afschrijving.

    `openclaw plugins inspect --all --runtime` rapporteert niet-gebundelde plugins waarvan laadfouten of
    diagnostiek nog naar deze verwijderde bestands-API's verwijzen. De adviserende
    controle van `@openclaw/plugin-inspector` moet versie `0.3.17` of nieuwer
    gebruiken, zodat scans van externe pakketten vóór een release ook helpers
    voor volledige sessieopslagen, helpers voor sessiebestandspaden, verouderde
    transcriptbestandsdoelen en laag-niveau-transcripthelpers markeren.

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **Oud**: `runtime.tasks.flow` (enkelvoud) retourneerde een live accessor voor
    taakstromen.

    **Nieuw**: `runtime.tasks.managedFlows` behoudt de beheerde TaskFlow-mutatie-runtime
    voor plugins die vanuit een stroom onderliggende taken maken, bijwerken,
    annuleren of uitvoeren. Gebruik `runtime.tasks.flows` wanneer de plugin alleen
    op DTO's gebaseerde leesbewerkingen nodig heeft.

    ```typescript
    // Voor
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // Na
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    De verouderde aliassen zijn in juli 2026 verwijderd.

  </Accordion>

  <Accordion title="Ingebedde extensiefabrieken -> middleware voor agenttoolresultaten">
    Hierboven behandeld in [Migreren](#how-to-migrate). Voor de
    volledigheid hier opgenomen: het verwijderde, uitsluitend voor de ingebedde runner bestemde
    pad `api.registerEmbeddedExtensionFactory(...)` is vervangen door
    `api.registerAgentToolResultMiddleware(...)` met een expliciete runtimelijst
    in `contracts.agentToolResultMiddleware`.
  </Accordion>

  <Accordion title="Alias OpenClawSchemaType -> OpenClawConfig">
    De root-SDK-alias `OpenClawSchemaType` is verwijderd. Gebruik de canonieke
    naam `OpenClawConfig`.

    ```typescript
    // Voor
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // Na
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
Verouderingen op extensieniveau (binnen gebundelde kanaal-/providerplugins onder
`extensions/`) worden bijgehouden in hun eigen `api.ts`- en `runtime-api.ts`-
barrels. Ze hebben geen invloed op contracten van plugins van derden en worden
hier niet vermeld. Als je rechtstreeks de lokale barrel van een gebundelde plugin gebruikt,
lees dan vóór het upgraden de opmerkingen over veroudering in die barrel.
</Note>

## Migratie van Talk en realtime-spraak

Code voor realtime-spraak, telefonie, vergaderingen en browser-Talk gebruikt één Talk-
sessiecontroller die wordt geëxporteerd door `openclaw/plugin-sdk/realtime-voice`. De
controller beheert de algemene Talk-eventenvelop, de status van de actieve beurt, de opname-
status, de status van uitgaande audio, de recente eventgeschiedenis en het weigeren van verouderde beurten.
Providerplugins beheren leveranciersspecifieke realtime-sessies. Plugins voor browservergaderingen
gebruiken `openclaw/plugin-sdk/meeting-runtime` voor sessie-, browser-, audio-, nodehost-,
agentconsultatie- en spraakoproepmechanismen en implementeren vervolgens `MeetingPlatformAdapter`
voor URL-regels, DOM-scripts, het koppelen van handmatige acties, ondertiteling, het aanmaken en inbel-
plannen. REST-API's van platforms, OAuth, artefacten, selectors en namen voor overdracht blijven in
de plugin. Browsermachtigingsplannen ontvangen de aangevraagde vergader-URL, zodat elk
platform alleen zijn exact ondersteunde origins kan toestaan. Sessieruntimes moeten ook
platformspecifieke live-status normaliseren nadat het verlaten van de browser is bevestigd;
historische transcriptvelden mogen behouden blijven, maar de gereedheid van ondertiteling en audio mag
na het verlaten niet actief blijven.

Alle gebundelde oppervlakken draaien op de gedeelde controller: browserrelay,
overdracht van beheerde ruimtes, realtime-spraakoproepen, streaming-STT voor spraakoproepen, realtime van Google
Meet en native push-to-talk. Gateway maakt één live Talk-eventkanaal
bekend in `hello-ok.features.events`: `talk.event`.

Nieuwe code hoort `createTalkEventSequencer(...)` niet rechtstreeks aan te roepen, tenzij
een adapter op laag niveau of testfixture wordt geïmplementeerd. Gebruik de gedeelde controller, zodat
events die aan een beurt zijn gekoppeld niet zonder beurt-id kunnen worden uitgezonden, verouderde aanroepen van `turnEnd` /
`turnCancel` een nieuwere actieve beurt niet kunnen wissen en lifecycle-events voor uitgaande audio
consistent blijven voor telefonie, vergaderingen, browserrelay,
overdracht van beheerde ruimtes en native Talk-clients.

De vorm van de openbare API:

```typescript
// Door Gateway beheerde Talk-sessie-API.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// Providersessie-API die door de client wordt beheerd.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

WebRTC-/provider-websocketsessies die door de browser worden beheerd, gebruiken `talk.client.create`,
omdat de browser de provideronderhandeling en het mediatransport beheert, terwijl de
Gateway de inloggegevens, instructies en het toolbeleid beheert. `talk.session.*` is
het algemene, door Gateway beheerde oppervlak voor realtime via gatewayrelay, transcriptie via gatewayrelay
en native STT/TTS-sessies in beheerde ruimtes.

Verouderde configuraties die realtime-selectors naast `talk.provider` /
`talk.providers` plaatsen, moeten worden gerepareerd met `openclaw doctor --fix`; Talk in de runtime
interpreteert providerconfiguratie voor spraak/TTS niet opnieuw als providerconfiguratie voor realtime.

De ondersteunde combinaties voor `talk.session.create` zijn bewust beperkt:

| Modus            | Transport       | Brein           | Beheerder          | Opmerkingen                                                                                                        |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Full-duplex provideraudio die via de Gateway wordt overbrugd; toolaanroepen worden via de agentconsultatietool gerouteerd. |
| `transcription` | `gateway-relay` | `none`          | Gateway            | Alleen streaming-STT; aanroepers sturen invoeraudio en ontvangen transcriptevents.                                 |
| `stt-tts`       | `managed-room`  | `agent-consult` | Native/clientruimte | Ruimtes in push-to-talk- en walkietalkiestijl waarin de client opname/afspelen beheert en de Gateway de beurtstatus beheert. |
| `stt-tts`       | `managed-room`  | `direct-tools`  | Native/clientruimte | Ruimtemodus uitsluitend voor beheerders, voor vertrouwde eigen oppervlakken die Gateway-toolacties rechtstreeks uitvoeren. |

Methodetoewijzing voor lezers die migreren van de oudere families `talk.realtime.*` /
`talk.transcription.*` / `talk.handoff.*` (allemaal verwijderd):

| Oud                              | Nieuw                                                    |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` of `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

Het uniforme besturingsvocabulaire is eveneens bewust beperkt:

| Methode                         | Van toepassing op                                       | Contract                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | Voeg een base64-PCM-audiofragment toe aan de providersessie die door dezelfde Gateway-verbinding wordt beheerd.                                                                                                           |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | Start een gebruikersbeurt in een beheerde ruimte.                                                                                                                                                                        |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | Beëindig de actieve beurt na validatie op een verouderde beurt.                                                                                                                                                           |
| `talk.session.cancelTurn`       | alle door Gateway beheerde sessies                       | Annuleer actieve opname/provider-/agent-/TTS-werkzaamheden voor een beurt.                                                                                                                                                |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | Stop de audio-uitvoer van de assistent zonder noodzakelijkerwijs de gebruikersbeurt te beëindigen.                                                                                                                        |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | Voltooi een providertoolaanroep na elke asynchrone voltooiing die door de bridge beschikbaar wordt gemaakt; geef `options.willContinue` door voor tussentijdse uitvoer of, indien ondersteund, `options.suppressResponse` om nog een assistentantwoord te voorkomen. |
| `talk.session.steer`            | door een agent ondersteunde Talk-sessies                  | Stuur gesproken besturing via `status`, `steer`, `cancel` of `followup` naar de actieve ingebedde uitvoering die vanuit de Talk-sessie is bepaald.                                  |
| `talk.session.close`            | alle uniforme sessies                                     | Stop relaysessies of trek de status van beheerde ruimtes in en vergeet vervolgens de uniforme sessie-id.                                                                                                                 |

Voer geen speciale gevallen voor providers of platforms in de kern in om dit te laten werken.
De kern beheert de semantiek van Talk-sessies. Providerplugins beheren het opzetten van leverancierssessies.
Spraakoproepen en Google Meet beheren telefonie-/vergaderadapters. Browser- en native
apps beheren de UX voor opname/afspelen op het apparaat.

## Tijdlijn voor verwijdering

| Wanneer                                     | Wat gebeurt er                                                                                                                              |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Nu**                                      | Verouderde oppervlakken die waarschuwingen ondersteunen, geven runtimewaarschuwingen; repositorycontroles weigeren verouderde SDK-imports vanuit de core en gebundelde plugins. |
| **In afwachting van beslissing van de eigenaar** | Records zonder datum blijven verouderd en komen niet in aanmerking voor verwijdering totdat hun eigenaar een `removeAfter`-datum publiceert. |
| **De `removeAfter`-datum van elk compatibiliteitsrecord** | Dat specifieke oppervlak komt in aanmerking voor verwijdering; `pnpm plugins:boundary-report --fail-on-eligible-compat` laat CI mislukken zodra de datum is verstreken. |
| **Volgende hoofdrelease**                   | Oppervlakken met een datum mogen pas na hun `removeAfter`-datum worden verwijderd; records zonder datum vereisen nog steeds goedkeuring van de eigenaar en een gepubliceerde datum. |

De resterende openbare SDK-subpaden hieronder hebben registergestuurde verwijderingsvensters.
De rijen van 30 juli zijn verwijderd na hun vroege, door onderhouders goedgekeurde opschoningsronde:
ongebruikte subpaden zijn verwijderd, eerdere compatibiliteitsaliassen zijn verwijderd en
uitsluitend gebundelde modules zijn gedegradeerd tot privé-lokale buildtoewijzingen.

| `removeAfter` | Niveau                            | SDK-subpaden                                                                                                                                                                         |
| ------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `2026-08-15`  | Eerdere compatibiliteitsverouderingen | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod` |
| `2026-09-01`  | Eerdere compatibiliteitsverouderingen | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                  |
| `2026-10-01`  | Verouderde mediaprojectie         | `agent-media-payload`, plus de niet-subpadvelden van `MsgContext Media*`, builders voor inkomende mediapayloads van kanalen, `buildMediaPayload`, media-aliassen voor hooks en `{{Media*}}`-sjablonen |

Alle coreplugins zijn al gemigreerd. Externe plugins moeten migreren
vóór de volgende hoofdrelease. Voer `pnpm plugins:boundary-report` uit om te zien welke
compatibiliteitsrecords het eerst verlopen voor de oppervlakken die jouw plugin gebruikt.

## De waarschuwingen tijdelijk onderdrukken

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Dit is een tijdelijke uitweg, geen permanente oplossing.

## Gerelateerd

- [Aan de slag](/nl/plugins/building-plugins) - bouw je eerste plugin
- [SDK-overzicht](/nl/plugins/sdk-overview) - volledige referentie voor imports van subpaden
- [Kanaalplugins](/nl/plugins/sdk-channel-plugins) - kanaalplugins bouwen
- [Providerplugins](/nl/plugins/sdk-provider-plugins) - providerplugins bouwen
- [Interne werking van plugins](/nl/plugins/architecture) - diepgaande architectuurbeschrijving
- [Pluginmanifest](/nl/plugins/manifest) - referentie voor het manifestschema
