---
read_when:
    - Je moet vanuit een plugin kernhulpfuncties aanroepen (TTS, STT, beeldgeneratie, zoeken op het web, Gateway, subagent, nodes)
    - Je wilt begrijpen wat api.runtime beschikbaar stelt
    - Je benadert configuratie-, agent- of mediahelpers vanuit plugincode
sidebarTitle: Runtime helpers
summary: api.runtime -- de geïnjecteerde runtimehelpers die beschikbaar zijn voor plugins
title: Runtimehelpers voor Plugins
x-i18n:
    generated_at: "2026-07-27T05:43:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

Naslag voor het `api.runtime`-object dat tijdens de registratie in elke plugin wordt geïnjecteerd. Gebruik deze helpers in plaats van interne onderdelen van de host rechtstreeks te importeren.

<CardGroup cols={2}>
  <Card title="Kanaalplugins" href="/nl/plugins/sdk-channel-plugins">
    Stapsgewijze handleiding die deze helpers in hun context gebruikt voor kanaalplugins.
  </Card>
  <Card title="Providerplugins" href="/nl/plugins/sdk-provider-plugins">
    Stapsgewijze handleiding die deze helpers in hun context gebruikt voor providerplugins.
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` is de huidige productversie van OpenClaw, afkomstig uit de gedeelde versie-resolver, zodat plugins dezelfde waarde zien als de CLI rapporteert.

## Configuratie laden en schrijven

Geef de voorkeur aan configuratie die al aan het actieve aanroeppad is doorgegeven, bijvoorbeeld `api.config` tijdens de registratie of een `cfg`-argument bij callbacks van kanalen/providers. Zo stroomt één processnapshot door het werk in plaats van configuratie opnieuw te parsen in veelgebruikte paden.

Gebruik `api.runtime.config.current()` alleen wanneer een langlevende handler de huidige processnapshot nodig heeft en er geen configuratie aan die functie is doorgegeven. De geretourneerde waarde is alleen-lezen; kloon deze of gebruik een mutatiehelper voordat je deze bewerkt.

Toolfactories ontvangen `ctx.runtimeConfig` plus `ctx.getRuntimeConfig()`. Gebruik de getter binnen de `execute`-callback van een langlevende tool wanneer de configuratie kan veranderen nadat de tooldefinitie is gemaakt.

Sla wijzigingen op met `api.runtime.config.mutateConfigFile(...)` of `api.runtime.config.replaceConfigFile(...)`. Elke schrijfbewerking moet een expliciet `afterWrite`-beleid kiezen:

- `afterWrite: { mode: "auto" }` laat de planner voor het herladen van de Gateway beslissen.
- `afterWrite: { mode: "restart", reason: "..." }` dwingt een schone herstart af wanneer de schrijver weet dat opnieuw laden tijdens runtime onveilig is.
- `afterWrite: { mode: "none", reason: "..." }` onderdrukt automatisch herladen/herstarten alleen wanneer de aanroeper verantwoordelijk is voor de vervolgstap.

De mutatiehelpers retourneren `afterWrite` plus een getypeerde `followUp`-samenvatting, zodat aanroepers kunnen loggen of testen of ze een herstart hebben aangevraagd. De Gateway bepaalt nog steeds wanneer die herstart daadwerkelijk plaatsvindt.

Gebruik `current()`, een doorgegeven `cfg`, `mutateConfigFile(...)` of
`replaceConfigFile(...)` voor toegang tot en schrijfbewerkingen van runtimeconfiguratie.

Geef voor rechtstreekse SDK-imports de voorkeur aan de gerichte configuratiesubpaden boven de brede `openclaw/plugin-sdk/config-runtime`-compatibiliteitsbarrel: `config-contracts` voor typen, `runtime-config-snapshot` voor actuele processnapshots en `config-mutation` voor schrijfbewerkingen. Lees waarden die aan een entry zijn gebonden uit `api.pluginConfig`; gebruik een aangeleverde toolcontext alleen voor de runtimebrede configuratiesnapshot en houd pluginspecifieke samenvoeging op die grens. Tests voor gebundelde plugins moeten deze gerichte subpaden rechtstreeks mocken in plaats van de brede compatibiliteitsbarrel te mocken.

Interne runtimecode van OpenClaw volgt dezelfde richting: laad configuratie één keer aan de grens van de CLI, Gateway of het proces en geef die waarde vervolgens door. Geslaagde mutatieschrijfbewerkingen vernieuwen de runtimeprocessnapshot en verhogen de interne revisie; langlevende caches moeten de runtime-eigen cachesleutel gebruiken in plaats van configuratie lokaal te serialiseren. Voor langlevende runtimemodules geldt een nultolerantiescanner voor omgevingsgebonden `loadConfig()`-aanroepen; gebruik een doorgegeven `cfg`, een aanvraag-`context.getRuntimeConfig()` of `getRuntimeConfig()` aan een expliciete procesgrens.

Uitvoeringspaden van providers en kanalen moeten de actieve runtimeconfiguratiesnapshot gebruiken, niet een bestandssnapshot die wordt geretourneerd voor het teruglezen of bewerken van configuratie. Bestandssnapshots behouden bronwaarden zoals SecretRef-markeringen voor de UI en schrijfbewerkingen; providercallbacks hebben de opgeloste runtimeweergave nodig. Wanneer een helper kan worden aangeroepen met de actieve bronsnapshot of de actieve runtimesnapshot, routeer dan via `selectApplicableRuntimeConfig()` voordat je inloggegevens leest.

## Herbruikbare runtimehulpprogramma's

Gebruik binnenkomende `botLoopProtection`-feiten voor binnenkomende berichten die door bots zijn opgesteld. Core past de gedeelde sliding-window-beveiliging in het geheugen toe vóór sessieregistratie en dispatch, zonder het beleid aan één kanaal te koppelen. De beveiliging volgt `(scopeId, conversationId, participant pair)`-sleutels, telt beide richtingen van een paar samen, past een afkoelperiode toe zodra het vensterbudget wordt overschreden en verwijdert inactieve entries wanneer de gelegenheid zich voordoet.

Kanaalplugins die dit gedrag beschikbaar stellen aan operators, moeten bij voorkeur de gedeelde `channels.defaults.botLoopProtection`-structuur voor basisbudgetten gebruiken en daar vervolgens kanaal-/providerspecifieke overrides bovenop leggen. De gedeelde configuratie gebruikt seconden omdat deze voor gebruikers zichtbaar is:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

Geef genormaliseerde feiten over het botpaar door met de opgeloste beurt. Core lost standaardwaarden, eenheidsconversie en `enabled`-semantiek op:

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

Gebruik `openclaw/plugin-sdk/pair-loop-guard-runtime` alleen rechtstreeks voor aangepaste
gebeurtenislussen tussen twee partijen die niet via de gedeelde runner voor binnenkomende antwoorden lopen.

## Runtime-naamruimten

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    Agentidentiteit, mappen en sessiebeheer.

    ```typescript
    // De werkmap van de agent bepalen (agentId is vereist)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // De agentwerkruimte bepalen
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // De agentidentiteit ophalen
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // Het standaard denkniveau ophalen
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // Een door de gebruiker opgegeven denkniveau valideren aan de hand van het actieve providerprofiel
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // niveau doorgeven aan een ingebedde uitvoering
    }

    // De time-out van de agent ophalen
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // Controleren of de werkruimte bestaat
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // Een ingebedde agentbeurt uitvoeren
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "Vat de meest recente wijzigingen samen",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` is de neutrale helper om vanuit plugincode een normale OpenClaw-agentbeurt te starten. Deze gebruikt dezelfde provider-/modelresolutie en selectie van de agentharness als antwoorden die door kanalen worden geactiveerd.

    `runEmbeddedPiAgent(...)` blijft bestaan als verouderde compatibiliteitsalias voor bestaande plugins. Nieuwe code moet `runEmbeddedAgent(...)` gebruiken.

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` deelt de dispatchbeslissing van de CLI-backend van de ingebedde runner (route, de door de backend gedeclareerde `subscriptionAuthDispatch`-mogelijkheid, opgeslagen modus voor inloggegevens — met inachtneming van een expliciet vastgezette `authProfileId`) met aanroepers die ingebedde uitvoeringen aanmelden voor `cliBackendDispatch: "subscription-auth"`. Het retourneert `{ provider }` wanneer de uitvoering via de CLI-backend zou plaatsvinden en `undefined` wanneer deze op de rechtstreekse passthrough blijft, zodat aanroepers time-outs kunnen begroten voor de uitvoering die daadwerkelijk zal plaatsvinden.

    `resolveThinkingPolicy(...)` retourneert de ondersteunde denkniveaus en optionele standaardwaarde van de provider/het model. Providerplugins beheren het modelspecifieke profiel via hun denkhooks, dus toolplugins moeten deze runtimehelper aanroepen in plaats van providerlijsten te importeren of dupliceren.

    `normalizeThinkingLevel(...)` zet gebruikerstekst zoals `on`, `x-high` of `extra high` om naar het canonieke opgeslagen niveau voordat dit aan het opgeloste beleid wordt getoetst.

    **Helpers voor de sessieopslag** staan onder `api.runtime.agent.session`:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // Sessierijen doorlopen zonder afhankelijk te zijn van de verouderde sessions.json-structuur.
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // De sessie maken of bijwerken en vervolgens signal doorgeven aan de toegelaten agentuitvoering.
      },
    );
    ```

    Geef voor sessieworkflows de voorkeur aan `getSessionEntry(...)`, `listSessionEntries(...)`, `patchSessionEntry(...)` of `upsertSessionEntry(...)`. Deze helpers adresseren sessies op basis van de agent-/sessie-identiteit, zodat plugins niet afhankelijk zijn van de verouderde `sessions.json`-opslagstructuur. Gebruik `preserveActivity: true` voor patches die alleen metadata bevatten en de sessieactiviteit niet mogen vernieuwen, en `replaceEntry: true` alleen wanneer de callback een volledige entry retourneert en verwijderde velden verwijderd moeten blijven. Doctor- en migratiepaden kunnen `fallbackEntry`, `skipMaintenance` en `requireWriteSuccess` combineren voor één atomair herstel van de canonieke opslag.

    `createSessionEntry(...)` maakt een nieuwe canonieke sessierij en transcriptie. Het vertrouwde `initialEntry`-oppervlak is bewust beperkt: een niet-lege `agentHarnessId`, optionele `modelSelectionLocked: true` en optionele `pluginExtensions`. De geïnjecteerde runtime accepteert via `registerAgentHarness(...)` alleen harness-id's die eigendom zijn van de aanroepende plugin; dit is een eigendomsinvariant, geen sandbox tussen plugins binnen hetzelfde proces. Een bestaande rij wordt geweigerd; `label` en `spawnedCwd` zijn afzonderlijke aanmaakvelden in plaats van patches voor vertrouwde entries.

    Tijdens het aanmaken blijft de mutatiebarrière voor de sessielevenscyclus via `afterCreate` actief, zodat nieuw werk wacht tot de initialisatie door de plugin is voltooid en reeds toegelaten werk ertoe leidt dat het aanmaken mislukt. De callback ontvangt een kloon van de aangemaakte status. Als deze een patch retourneert, mag die patch alleen `pluginExtensions` bevatten en is de waarde daarvan het volledige definitieve `pluginExtensions`-veld. Bij een fout in de callback of de definitieve persistentie worden de ongewijzigde nieuwe rij en transcriptie teruggedraaid; beveiligd terugdraaien behoudt een rij die gelijktijdig is gewijzigd of geclaimd. `recoverMatchingInitialEntry: true` is alleen bedoeld om een onderbroken initialisatie opnieuw te proberen wanneer de opgeslagen vertrouwde velden exact overeenkomen, en herstel vereist dat `afterCreate` een definitieve patch retourneert.

    Gebruik `runWithWorkAdmission(...)` wanneer een plugin werk start op een opgeslagen sessie. De callback weigert gearchiveerde of gelijktijdig vervangen sessies, houdt archiverings-, reset- en verwijderingsmutaties gecoördineerd tot voltooiing en ontvangt een `AbortSignal` die aan de agentuitvoering moet worden doorgegeven. Een harness kan via het experimentele registratieveld `delegatedExecutionPluginIds` expliciet vertrouwde uitvoeringsgedelegeerden benoemen. Gedelegeerden kunnen alleen een exact bestaande, modelvergrendelde sessie toelaten en uitvoeren; alle sessiemutaties blijven beperkt tot de eigenaar van de harness. Zie [Agentharness-plugins](/nl/plugins/sdk-agent-harness#delegated-execution).

    Onderhouds- en reparatieplugins kunnen `deleteSessionEntry(...)` gebruiken voor één sessie-item binnen een bepaald bereik, `cleanupSessionLifecycleArtifacts(...)` voor door de levenscyclus beheerde tijdelijke sessies en `resolveSessionStoreBackupPaths(...)` voordat een opslag wordt gewijzigd. Geef `expectedSessionId` en `expectedUpdatedAt` door wanneer verwijdering niet mag conflicteren met een gelijktijdige sessie-update; gebruik `expectedSessionId: null` wanneer de eerdere momentopname geen sessie-id had. Deze helpers zijn beperkte oppervlakken voor reparatie en levenscyclusbeheer, geen algemene API voor het verwijderen uit een opslag.

    `resolveStorePath(...)` en `updateSessionStoreEntry(...)` completeren de sessiehelpers: `resolveStorePath` bepaalt het pad naar de sessieopslag voor een bepaald bereik en `updateSessionStoreEntry({ storePath, sessionKey, update })` past één item rechtstreeks aan via het opslagpad wanneer de aanroeper dit al kent.

    `loadTranscriptEventsSync(...)` is beschikbaar voor synchrone doctor- en reparatiepaden die de asynchrone transcriptruntime niet kunnen gebruiken. Deze retourneert onbewerkte `SessionStoreTranscriptEvent`-records. Normale runtimecode van plugins hoort bij voorkeur `openclaw/plugin-sdk/session-transcript-runtime` te gebruiken.

    `formatSqliteSessionFileMarker(...)`, `parseSqliteSessionFileMarker(...)` en `sqliteSessionFileMarkerMatchesSession(...)` zijn overgangshelpers voor code die nog steeds een verouderd veld met de naam `sessionFile` ontvangt. Een geparseerde SQLite-markering identificeert een actief SQLite-transcriptdoel; het is geen bestandssysteempad. Nieuwe API's horen getypeerde sessie-identiteit te gebruiken in plaats van markeringsreeksen.

    Importeer voor het lezen en schrijven van transcripties `openclaw/plugin-sdk/session-transcript-runtime` en gebruik `resolveSessionTranscriptIdentity(...)`, `resolveSessionTranscriptTarget(...)`, `readSessionTranscriptEvents(...)`, `readSessionTranscriptRawDelta(...)`, `readSessionTranscriptVisibleMessageDelta(...)`, `readVisibleSessionTranscriptMessageEntries(...)`, `appendSessionTranscriptMessageByIdentity(...)`, `publishSessionTranscriptUpdateByIdentity(...)` of `withSessionTranscriptWriteLock(...)` met `{ agentId, sessionKey, sessionId }`. Met deze API's kunnen plugins een transcript identificeren, onbewerkte gebeurtenissen of zichtbare vertakkingsveilige berichtitems lezen, berichten toevoegen, updates publiceren en gerelateerde bewerkingen uitvoeren onder dezelfde schrijfvergrendeling van het transcript, zonder afhankelijk te zijn van actieve transcriptbestandspaden. `readVisibleSessionTranscriptMessageEntries(...)` retourneert geordende leesmetagegevens; het veld `seq` daarvan is geen hervatbare cursor.

    `appendSessionTranscriptMessageByIdentity(...)` is een low-level toevoeging van een reeds canoniek bericht. Plugins mogen geen gebruikersrijen met media samenstellen met `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType` of `MediaTypes` op het hoogste niveau. Binnenkomende kanaalgegevens horen geordende feiten via `MsgContext.media` door te geven en de host de persistentie van gebruikersbeurten te laten beheren. Een door de host voorbereid persistent gebruikersbericht bevat canonieke geordende feiten onder `message.__openclaw.media`; de algemene toevoeg-API leidt verouderde parallelle arrays niet af en repareert ze niet.

    `readSessionTranscriptRawDelta(...)` retourneert een begrensd resultaat van het type `page`, `reset` of `missing`. Geef de ondoorzichtige `page.cursor` door aan de volgende aanroep. Zuivere toevoegingen behouden de cursor, terwijl vervanging van het transcript `reset` retourneert met een nieuwe opstartcursor. Pagina's bevatten standaard 1,000 gebeurtenissen en 1,000,000 geserialiseerde bytes; aanroepers kunnen maximaal 10,000 gebeurtenissen en 64 MiB aanvragen. Wanneer alleen de volgende gebeurtenis al groter is dan `maxBytes`, is de pagina leeg en rapporteert deze `requiredBytes`; probeer het opnieuw met ten minste die bytelimiet wanneer deze niet groter is dan 64 MiB. Grotere afzonderlijke gebeurtenissen vereisen de API voor volledig lezen. Een cursor identificeert alleen een positie en verleent nooit toegang tot een andere sessie.

    `readSessionTranscriptVisibleMessageDelta(...)` biedt dezelfde begrensde opstart-en-hervatstructuur voor de door de host beheerde actieve berichtprojectie. Deze retourneert berichten van oud naar nieuw, zodat contextengines de initiële geschiedenis kunnen verwerken en de ondoorzichtige cursor als hun watermerk kunnen opslaan. Sla de cursor ongewijzigd op en retourneer deze ongewijzigd; het is een aanwijzing om door te gaan, geen autorisatiereferentie. Lineaire toevoegingen worden hervat na het laatst geretourneerde bericht. Vervanging van het transcript, een cursor waarvan het anker de actieve vertakking heeft verlaten of daarbinnen is verplaatst, onjuist gevormde cursors en cursors van andere sessies retourneren `reset` met een nieuwe opstartcursor. De standaardwaarden en limieten voor aantal en bytes komen overeen met de onbewerkte delta-API. Terwijl de actieve projectie na een wijziging van vertakking opnieuw wordt opgebouwd, is het resultaat `unavailable` met reden `projection_rebuilding`; probeer het later opnieuw in plaats van terug te vallen op een actief transcriptbestand.

    De verouderde helpers voor de volledige opslag en actieve transcriptbestanden worden niet meer geëxporteerd vanuit de plugin-SDK. Gebruik de helpers voor items binnen een bepaald bereik voor sessiemetagegevens en de helpers voor transcriptidentiteit voor bewerkingen op actieve transcripties. Archief- en ondersteuningsworkflows die bestandsartefacten nodig hebben, horen hun eigen archiefoppervlakken te gebruiken in plaats van runtime-API's voor actieve sessies.

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    Standaardconstanten voor model en provider:

    ```typescript
    const model = api.runtime.agent.defaults.model; // bijv. "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // bijv. "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    Voer een door de host beheerde tekstaanvulling uit zonder interne provideronderdelen te importeren of
    de voorbereiding van OpenClaw voor model, authenticatie en basis-URL te dupliceren.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "Vat dit transcript samen." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    Providerorkestratie kan ook de geconfigureerde levenscyclus van de lokale service
    verkrijgen voordat een HTTP-aanvraag wordt verzonden:

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // Verzend en verwerk de provideraanvraag volledig.
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` is een stabiel, algemeen SDK-contract voor
    providerservices. De host bepaalt de procesconfiguratie vanuit
    `models.providers.<providerId>.localService`; aanroepers kunnen geen
    opdracht, argumenten, omgeving of levenscyclusbeleid opgeven. Het starten van processen,
    gereedheidscontrole, diagnostiek en het beleid voor stoppen bij inactiviteit blijven intern voor de host.

    Geef de exact geconfigureerde provider-id en de bepaalde basis-URL voor aanvragen door. Vervang
    aliassen niet door een adapter-id: afzonderlijke aliassen kunnen naar afzonderlijke
    lokale GPU-hosts verwijzen. De host weigert eindpunten die niet overeenkomen met de geconfigureerde
    basis-URL van de provider, afgezien van de normalisatie `/v1` die door Ollama- en LM
    Studio-adapters wordt gebruikt. De host beheert serialisatie bij het opstarten, gereedheidscontroles,
    aanvraagleases, afbreekverwerking en afsluiting bij inactiviteit.

    De helper gebruikt hetzelfde eenvoudige voorbereidingspad voor aanvullingen als de
    ingebouwde runtime van OpenClaw en de door de host beheerde momentopname van de runtimeconfiguratie. Contextengines
    ontvangen een aan een sessie gebonden `llm.complete`-mogelijkheid, zodat modelaanroepen de
    agent van de actieve sessie gebruiken en niet stilzwijgend terugvallen op de standaardagent. Het
    resultaat bevat toeschrijving aan provider, model en agent, plus genormaliseerd token-,
    cache- en geschat kostengebruik wanneer beschikbaar.

    Stel `reasoning` in om een redeneerinspanning voor het geselecteerde model aan te vragen. De
    host normaliseert de canonieke denkniveaus (`off`, `minimal`, `low`,
    `medium`, `high`, `xhigh`, `adaptive`, `max` en `ultra`) voor de geselecteerde
    provider en het geselecteerde model voordat de aanvulling wordt verzonden. `adaptive` wordt
    `medium`; `max` en `ultra` worden `max` indien ondersteund, en anders `xhigh`.

    <Warning>
    Modeloverschrijvingen vereisen dat de operator zich hiervoor aanmeldt via `plugins.entries.<id>.llm.allowModelOverride: true` in de configuratie. Gebruik `plugins.entries.<id>.llm.allowedModels` om vertrouwde plugins te beperken tot specifieke canonieke `provider/model`-doelen. Aanvullingen tussen agents vereisen `plugins.entries.<id>.llm.allowAgentIdOverride: true`.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    Roep een andere Gateway-methode binnen hetzelfde proces aan met behoud van de vertrouwde runtime-
    identiteit van de huidige plugin. Dit is bedoeld voor gebundelde of vertrouwde officiële plugins die plugin-eigen
    Gateway-mogelijkheden combineren zonder een WebSocket-loopbackverbinding te openen.

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    Aanvragen gebruiken het bereik `operator.write` en verlenen geen beheerdersbereik. Aanroepen van willekeurige externe
    plugins worden geweigerd. Mislukte methoden werpen een `GatewayClientRequestError` en behouden gestructureerde
    `details`, metagegevens voor opnieuw proberen en de Gateway-foutcode voor herstelworkflows. Gebruik `isAvailable()`
    voordat je dit pad kiest vanuit tools die ook in zelfstandige agentprocessen kunnen worden uitgevoerd.

  </Accordion>
  <Accordion title="api.runtime.subagent">
    Start en beheer subagentuitvoeringen op de achtergrond.

    ```typescript
    // Start een subagentuitvoering
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "Werk deze zoekopdracht uit tot gerichte vervolgzoekopdrachten.",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // optionele overschrijving
      model: "gpt-5.6-sol", // optionele overschrijving
      deliver: false,
    });

    // Wacht op voltooiing
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // Lees sessieberichten
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // Verwijder een sessie
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    Modeloverschrijvingen (`provider`/`model`) vereisen dat de operator zich hiervoor aanmeldt via `plugins.entries.<id>.subagent.allowModelOverride: true` in de configuratie. Niet-vertrouwde plugins kunnen nog steeds subagents uitvoeren, maar overschrijvingsaanvragen worden geweigerd.
    </Warning>

    `toolsAlsoAllow` voegt exacte, uniek beheerde tools die door de aanroepende plugin zijn geregistreerd toe aan het normale tooloppervlak van de worker. De runtime weigert kerntools en namen die met een andere plugin worden gedeeld. Profielen en toolbeleid van de operator blijven van toepassing, inclusief expliciete toelatingslijsten en weigeringen.

    `deleteSession(...)` kan sessies verwijderen die door dezelfde plugin via `api.runtime.subagent.run(...)` zijn aangemaakt. Voor het verwijderen van willekeurige gebruikers- of operatorsessies blijft een Gateway-aanvraag met beheerdersbereik vereist.

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    Inspecteer de effectieve bevoegdheid voor de sandboxwerkruimte van een agentsessie.

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    Het resultaat meldt of deze sessie in een sandbox wordt uitgevoerd, of de werkruimte
    niet beschikbaar, alleen-lezen of beschrijfbaar is, en een optionele `confinementError`
    wanneer het effectieve Docker-, tool-, sessie-, browser- of verhoogde beleid aan
    die werkruimte kan ontsnappen. Gebruik dit voor door de host beheerde delegatiebeslissingen die
    een worker niet meer bevoegdheden mogen verlenen dan de aanroeper heeft. Het is een attestatiehelper,
    geen vervanging voor het controleren van de eigen autorisatie van de aanroeper.

    `prepareWorkspaceAuthority(...)` voert dezelfde beleidscontrole uit en
    bereidt ook de Docker-sandbox voor `workspaceDir` voor. Deze weigert een actieve container
    waarvan de hash van de actuele configuratie niet overeenkomt met de aangevraagde koppelingen of het beleid. Geef
    alleen exacte toolnamen door waarvan de aanroepende plugin de geregistreerde implementaties
    begrenst; jokertekenvoorvoegsels bewijzen geen eigendom van tools.

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Geef verbonden Nodes weer en roep een opdracht voor een Node-host aan vanuit door de Gateway geladen plugincode of vanuit CLI-opdrachten van een plugin. Gebruik dit wanneer een plugin lokaal werk op een gekoppeld apparaat beheert, bijvoorbeeld een browser- of audiobrug op een andere Mac.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` bevat de geadverteerde
    `nodePluginTools`-beschrijvingen van elke verbonden Node wanneer die Node door plugins of MCP ondersteunde
    tools beschikbaar stelt aan de agent. Deze beschrijvingen vertegenwoordigen de actuele verbindingsstatus: de Gateway
    verwijdert ze wanneer de verbinding met de Node wordt verbroken, en een Node kan ze vervangen door
    `node.pluginTools.update` nadat de lokale plugin-/MCP-inventaris is gewijzigd.

    Binnen de Gateway draait deze runtime in hetzelfde proces. In CLI-opdrachten van plugins roept deze de geconfigureerde Gateway aan via RPC, zodat opdrachten zoals `openclaw googlemeet recover-tab` gekoppelde Nodes vanuit de terminal kunnen inspecteren. Node-opdrachten verlopen nog steeds via de normale Node-koppeling van de Gateway, lijsten met toegestane opdrachten, beleidsregels voor Node-aanroepen door plugins en de lokale opdrachtafhandeling van de Node.

    Plugins die door Nodes gehoste agenttools beschikbaar stellen, kunnen `agentTool.defaultPlatforms` instellen voor ongevaarlijke opdrachten die standaard moeten worden toegestaan. Laat dit weg wanneer operators zich expliciet moeten aanmelden met `gateway.nodes.commands.allow`. Gevaarlijke opdrachten op Node-hosts moeten een beleid voor Node-aanroepen registreren met `api.registerNodeInvokePolicy(...)`; dit beleid wordt uitgevoerd in de Gateway nadat de lijst met toegestane opdrachten is gecontroleerd en voordat de opdracht naar de Node wordt doorgestuurd, zodat rechtstreekse `node.invoke`-aanroepen, door Nodes gehoste plugintools en plugintools op hoger niveau hetzelfde handhavingstraject delen.

    <Warning>
    Het optionele veld `scopes` vraagt Gateway-operatorscopes aan voor de aanroep. OpenClaw honoreert dit alleen voor meegeleverde plugins en vertrouwde installaties van officiële plugins; aanvragen van andere plugins verhogen de rechten van de aanroep niet. Gebruik dit alleen wanneer een vertrouwde plugin een Node-opdracht moet aanroepen met een strengere Gateway-scope, zoals `operator.admin`.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Koppel de status van Task Flow en Task Run aan een bestaande OpenClaw-sessiesleutel of vertrouwde toolcontext.

    - `api.runtime.tasks.managedFlows` kan mutaties uitvoeren: Task Flows maken, voortzetten en annuleren.
    - `api.runtime.tasks.flows` en `api.runtime.tasks.runs` zijn alleen-lezen DTO-weergaven voor lijsten en statuszoekopdrachten; beide stellen `bindSession(...)` / `fromToolContext(...)` beschikbaar, plus `get`, `list`, `findLatest` en `resolve`.

    Task Flow houdt duurzame workflowstatus met meerdere stappen bij. Het is geen planner:
    gebruik Cron of `api.session.workflow.scheduleSessionTurn(...)` voor toekomstige
    activeringen en gebruik vervolgens `managedFlows` vanuit de geplande beurt wanneer dat werk
    flowstatus, onderliggende taken, wachttijden of annulering nodig heeft.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Nieuwe pull requests beoordelen",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "PR #123 beoordelen",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    Gebruik `bindSession({ sessionKey, requesterOrigin })` wanneer je al een vertrouwde OpenClaw-sessiesleutel uit je eigen bindingslaag hebt. Koppel niet op basis van onbewerkte gebruikersinvoer.

  </Accordion>
  <Accordion title="api.runtime.tts">
    Tekst-naar-spraaksynthese.

    ```typescript
    // Standaard-TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "Hallo vanuit OpenClaw",
      cfg: api.config,
    });

    // Voor telefonie geoptimaliseerde TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "Hallo vanuit OpenClaw",
      cfg: api.config,
    });

    // Beschikbare stemmen weergeven
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    Gebruikt de `tts`-configuratie en providerselectie van de kern. Retourneert een PCM-audiobuffer en samplefrequentie. `textToSpeechStream` is ook beschikbaar voor streamingsynthese.

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    Analyse van afbeeldingen, audio en video.

    ```typescript
    // Een afbeelding beschrijven
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // Audio transcriberen
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // optioneel, wanneer MIME niet kan worden afgeleid
    });

    // Een video beschrijven
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // Algemene bestandsanalyse
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // Gestructureerde afbeeldingsextractie via een specifieke provider/modelcombinatie.
    // Neem ten minste één afbeelding op; tekstinvoer dient als aanvullende context.
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "Geef de voorkeur aan het afgedrukte totaal boven handgeschreven notities." },
      ],
      instructions: "Extraheer leverancier, totaal en doorzoekbare tags.",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    Retourneert `{ text: undefined }` wanneer geen uitvoer wordt geproduceerd (bijvoorbeeld bij overgeslagen invoer).

    `describeImageFileWithModel(...)` beschrijft een reeds bekende afbeelding via een specifieke provider/modelcombinatie en omzeilt daarmee de standaardresolutie van het actieve model die `describeImageFile(...)` gebruikt.

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    Afbeeldingen genereren.

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "Een robot die een zonsondergang schildert",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    Video's genereren, volgens dezelfde structuur als het genereren van afbeeldingen.

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "Een droneopname die bij zonsopgang over een kustlijn vliegt",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    Muziek genereren, volgens dezelfde structuur als het genereren van afbeeldingen.

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "Een opgewekte lo-fi-track voor een programmeersessie",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Zoeken op het web.

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw plugin-SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    Mediahulpprogramma's op laag niveau.

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "afbeelding"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    Momentopname van de huidige runtimeconfiguratie en transactionele configuratieschrijfbewerkingen. Geef de voorkeur aan
    configuratie die al aan het actieve aanroeppad is doorgegeven; gebruik
    `current()` alleen wanneer de handler de momentopname van het proces rechtstreeks nodig heeft.

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` en `replaceConfigFile(...)` retourneren een `followUp`-
    waarde, bijvoorbeeld `{ mode: "restart", requiresRestart: true, reason }`,
    die de intentie van de schrijver vastlegt zonder de controle over opnieuw opstarten aan de
    Gateway te ontnemen.

  </Accordion>
  <Accordion title="api.runtime.system">
    Hulpprogramma's op systeemniveau.

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // Verouderde compatibiliteitsalias.
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` voert onmiddellijk één Heartbeat-cyclus uit en omzeilt daarbij de normale samenvoegtimer. Geef `{ heartbeat: { target: "last" } }` door om aflevering naar het laatst actieve kanaal af te dwingen in plaats van de standaardonderdrukking via `target: "none"`.

    `runCommandWithTimeout(...)` retourneert vastgelegde `stdout` en `stderr`, optionele
    aantallen afkappingen, `code`, `signal`, `killed`, `termination` en
    `noOutputTimedOut`. Resultaten van een time-out en een time-out wegens ontbrekende uitvoer rapporteren `code: 124`
    wanneer het onderliggende proces geen afsluitcode anders dan nul levert. Afsluitingen door een signaal
    zonder time-out kunnen nog steeds `code: null` retourneren, dus gebruik `termination` en
    `noOutputTimedOut` om de redenen voor time-outs te onderscheiden.

  </Accordion>
  <Accordion title="api.runtime.events">
    Abonnementen op gebeurtenissen.

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    Logboekregistratie.

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    Resolutie van model- en providerauthenticatie.

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // Verificatiegegevens gereed voor aanvragen, inclusief runtime-uitwisselingen van providers (bijv. OAuth-vernieuwing)
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    Resolutie van de statusmap en door SQLite ondersteunde opslag met sleutels.

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binaire of tekstuele payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    Opslag met sleutels blijft na herstarts behouden en wordt geïsoleerd op basis van de aan de runtime gebonden Plugin-id. Gebruik `registerIfAbsent(...)` voor atomische deduplicatieclaims: dit retourneert `true` wanneer de sleutel ontbrak of verlopen was en is geregistreerd, of `false` wanneer er al een actieve waarde bestaat, zonder de waarde, aanmaaktijd of TTL ervan te overschrijven. Gebruik `deleteIf(...)` wanneer bij opschoning alleen de eerder waargenomen waarde mag worden verwijderd; het synchrone predicaat en de verwijdering worden in één SQLite-transactie uitgevoerd. Limieten: `maxEntries` per naamruimte, 50,000 actieve rijen per Plugin, JSON-waarden kleiner dan 64KB en optionele TTL-vervaltermijnen. Standaard verwijdert een schrijfbewerking bij het bereiken van een van beide rijlimieten de oudste actieve rijen uit de naamruimte waarnaar wordt geschreven; andere naamruimten worden voor die schrijfbewerking niet verwijderd en de schrijfbewerking mislukt alsnog als de naamruimte onvoldoende rijen kan vrijmaken. Stel `overflowPolicy: "reject-new"` in voor duurzame eigendomsrecords die nooit mogen worden verwijderd: nieuwe sleutels mislukken bij een van beide limieten, terwijl bestaande sleutels kunnen worden bijgewerkt.

    `openSyncKeyedStore<T>(...)` retourneert dezelfde opslagvorm met synchrone methoden (`register`, `registerIfAbsent`, `deleteIf`, `lookup`, `consume`, `clear` retourneren allemaal rechtstreeks waarden in plaats van promises) voor aanroepers die niet kunnen wachten met await.

    `openBlobStore<TMetadata>(...)` slaat begrensde binaire payloads op in gedeelde SQLite zonder base64 of begeleidende bestanden. Hiervoor zijn limieten vereist voor bytes per item, bytes per naamruimte en rijen; byte-arrays worden aan de API-grens gekopieerd; en metadata worden weergegeven zonder elke BLOB te laden. `register(...)` is een expliciete upsert, ook voor verlopen sleutels. `registerIfAbsent(...)` biedt botsingsveilige aanmaak: een verlopen sleutel blijft bezet totdat de eigenaar deze claimt met `deleteExpiredKey(key)` of `deleteExpired()`, waardoor de metadata behouden blijven die nodig zijn om gerelateerde benoemde artefacten na de SQLite-commit te verwijderen. Elke rij met een TTL is tijdelijk en wordt uitgesloten van back-ups en herstel, zelfs voordat deze verloopt; laat TTL weg voor duurzame, herstelbare status. Hostzekeringen beperken elke BLOB tot 100 MiB, elke Plugin tot 512 MiB aan fysiek opgeslagen BLOB's en elke Plugin tot 50,000 fysiek opgeslagen rijen, inclusief verlopen rijen die wachten op opschoning door de eigenaar. Gebruik `registerIfAbsent(...)` met `overflowPolicy: "reject-new"` wanneer externe materialisaties niet ongemerkt verweesd mogen raken door vervanging of verwijdering.

    `openChannelIngressQueue<TPayload>(...)` opent een persistente ingangswachtrij met het bereik van de aanroepende Plugin, voor het bufferen van inkomende gebeurtenissen die minstens één keer moeten worden verwerkt, ook na herstarts. Wanneer voor herstel van verouderde claims `shouldRecover` wordt gebruikt, geef dan ook `shouldRecoverCorrupt` op als beschadigde geclaimde payloads in quarantaine moeten worden geplaatst: dankzij de claimidentiteit die onafhankelijk is van de payload kan de Plugin het actieve eigenaars- en rijstrookbeleid behouden voordat de wachtrij de rij met een tombstone markeert.

    `withLease(...)` serialiseert samenwerkend Plugin-werk tussen OpenClaw-processen. Kies `database: { scope: "shared" }` voor één globale eigenaar of `{ scope: "agent", agentId }` voor onafhankelijk eigendom per agent. Geef `AbortSignal` van de callback door aan elke bewerking die kan mislukken. `assertOwned()` is een controlepunt op een specifiek moment voordat een volgende belangrijke stap wordt gestart; de host verifieert het eigendom ook na de callback. Verlies van de lease of annulering door de aanroeper breekt het signaal af. Wachten op verkrijging en Heartbeats vinden buiten korte synchrone SQLite-transacties plaats; plugins ontvangen nooit databasepaden of -handles. Dit is coöperatieve annulering, geen fencing-token of autorisatie voor externe schrijfbewerkingen zonder fencing.

    `openChannelIngressDrain(...)` opent de kanaalonafhankelijke kernworker boven die wachtrij (of maakt een wachtrij als er geen is opgegeven). Het leegloopproces beheert herstel van verouderde claims, serialisatie van claims per rijstrook, voltooiing bij overname of voltooiing bij terugkeer van de verzending, afhandeling voor opnieuw proberen/dode berichten, optionele vervanging vóór overname en een time-out voor stilstand tussen claim en overname. Verbind claimeigendom met het genereren van antwoorden via `turnAdoptionLifecycle` (via `bindIngressLifecycleToReplyOptions` uit `plugin-sdk/channel-outbound`). Kanaalplugins behouden het in de wachtrij plaatsen aan de acceptatiezijde, het afleiden van rijstroken, de classificatie als niet opnieuw uitvoerbaar en eventueel beleid voor autorisatie van vervanging.

    <Warning>
    `openBlobStore`, `openKeyedStore`, `openSyncKeyedStore`, `withLease`, `openChannelIngressQueue` en `openChannelIngressDrain` zijn in deze release alleen beschikbaar voor gebundelde plugins en vertrouwde installaties van officiële plugins.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    Kanaalspecifieke runtimehelpers (beschikbaar wanneer een kanaalplugin is geladen). Gegroepeerd op aandachtspunt:

    | Groep | Doel |
    | --- | --- |
    | `text` | Opdelen (`chunkText`, `chunkMarkdownText`, `resolveChunkMode`), detectie van besturingsopdrachten, conversie van Markdown-tabellen. |
    | `reply` | Verzending van antwoorden in gebufferde blokken, envelopopmaak, resolutie van effectieve configuratie voor berichten/menselijke vertraging. |
    | `routing` | `buildAgentSessionKey`, `resolveAgentRoute`. |
    | `pairing` | `buildPairingReply`, lezen/verwijderen van toelatingslijsten, upserts van koppelingsverzoeken en uit verzoeken afgeleide goedkeuringsitems. |
    | `media` | Externe media downloaden/opslaan (zie hieronder). |
    | `activity` | Laatste kanaalactiviteit vastleggen/lezen. |
    | `session` | Sessiemetadata uit inkomende gebeurtenissen, updates van de laatste route. |
    | `mentions` | Helpers voor vermeldingsbeleid (zie hieronder). |
    | `reactions` | Handles voor bevestigingsreacties als indicatoren van actieve verwerking. |
    | `groups` | Resolutie van groepsbeleid en verplichte vermeldingen. |
    | `debounce` | Debouncing van inkomende berichten. |
    | `commands` | Opdrachtautorisatie en gating van tekstopdrachten. |
    | `outbound` | De uitgaande adapter van een kanaal laden. |
    | `inbound` | Context voor inkomende gebeurtenissen bouwen en de gedeelde kern voor inkomende gebeurtenissen/antwoorden uitvoeren. |
    | `threadBindings` | Time-out voor inactiviteit/maximale leeftijd voor gebonden sessiethreads aanpassen. |
    | `runtimeContexts` | Proceslokale context per kanaal/account/capaciteit registreren, lezen en bewaken. |

    `api.runtime.channel.media` is het aanbevolen oppervlak voor het downloaden en opslaan van kanaalmedia:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    Gebruik `saveRemoteMedia(...)` wanneer een externe URL OpenClaw-media moet worden. Gebruik `saveResponseMedia(...)` wanneer de Plugin al een `Response` heeft opgehaald met door de Plugin beheerde verificatiegegevens, omleidingsafhandeling of toelatingslijstafhandeling. Gebruik `readRemoteMediaBuffer(...)` alleen wanneer de Plugin onbewerkte bytes nodig heeft voor inspectie, transformaties, ontsleuteling of opnieuw uploaden. `fetchRemoteMedia(...)` blijft een verouderde compatibiliteitsalias voor `readRemoteMediaBuffer(...)`.

    `api.runtime.channel.mentions` is het gedeelde oppervlak voor beleid rond inkomende vermeldingen voor gebundelde kanaalplugins die runtime-injectie gebruiken:

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    Beschikbare vermeldingshelpers:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    Gebruik het genormaliseerde `{ facts, policy }`-pad voor beslissingen over vermeldingen.

    Verschillende velden onder `reply`, `session` en `inbound` bevatten per veld `@deprecated`-notities die verwijzen naar de huidige kern voor kanaalbeurten of uitgaande kanaaladapters; controleer de inline-JSDoc van de specifieke helper voordat je er nieuwe code op bouwt.

  </Accordion>
</AccordionGroup>

## Runtimeverwijzingen opslaan

Gebruik `createPluginRuntimeStore` om de runtimeverwijzing op te slaan voor gebruik buiten de `register`-callback:

<Steps>
  <Step title="De opslag maken">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "runtime van my-plugin is niet geïnitialiseerd",
    });
    ```

  </Step>
  <Step title="Met het toegangspunt verbinden">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "Mijn Plugin",
      description: "Voorbeeld",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="Vanuit andere bestanden openen">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // genereert een fout als deze niet is geïnitialiseerd
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // retourneert null als deze niet is geïnitialiseerd
    }
    ```

  </Step>
</Steps>

<Note>
Geef de voorkeur aan `pluginId` voor de identiteit van de runtimeopslag. De vorm op lager niveau, `key`, is bedoeld voor ongebruikelijke gevallen waarin één Plugin bewust meer dan één runtimeslot nodig heeft.
</Note>

## Andere `api`-velden op het hoogste niveau

Naast `api.runtime` biedt het API-object ook:

<ParamField path="api.id" type="string">
  Plugin-id.
</ParamField>
<ParamField path="api.name" type="string">
  Weergavenaam van de Plugin.
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  Huidige configuratiesnapshot (actieve in-memory runtimesnapshot indien beschikbaar).
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  Pluginspecifieke configuratie uit `plugins.entries.<id>.config`.
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  Logger met beperkt bereik (`debug`, `info`, `warn`, `error`).
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  Huidige laadmodus: `"full"` (live-activering), `"discovery"` / `"tool-discovery"` (alleen-lezen detectie van mogelijkheden), `"setup-only"` (lichtgewicht setup-ingang), `"setup-runtime"` (setupflow waarvoor ook de runtimekanaalingang nodig is), of `"cli-metadata"` (verzameling van metadata voor CLI-opdrachten).
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Los een pad op ten opzichte van de hoofdmap van de Plugin.
</ParamField>

## Gerelateerd

- [Interne werking van Plugins](/nl/plugins/architecture) — mogelijkhedenmodel en register
- [SDK-ingangspunten](/nl/plugins/sdk-entrypoints) — opties voor `definePluginEntry`
- [SDK-overzicht](/nl/plugins/sdk-overview) — subpadreferentie
