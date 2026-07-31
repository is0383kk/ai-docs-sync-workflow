---
read_when:
    - Sie müssen zentrale Hilfsfunktionen aus einem Plugin aufrufen (TTS, STT, Bildgenerierung, Websuche, Gateway, Subagent, Nodes)
    - Sie möchten verstehen, was `api.runtime` bereitstellt
    - Sie greifen aus Plugin-Code auf Konfigurations-, Agenten- oder Medien-Hilfsfunktionen zu
sidebarTitle: Runtime helpers
summary: api.runtime -- die injizierten Laufzeit-Hilfsfunktionen, die Plugins zur Verfügung stehen
title: Hilfsfunktionen für die Plugin-Laufzeit
x-i18n:
    generated_at: "2026-07-26T18:31:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

Referenz für das Objekt `api.runtime`, das bei der Registrierung in jedes Plugin injiziert wird. Verwenden Sie diese Hilfsfunktionen, anstatt Host-Interna direkt zu importieren.

<CardGroup cols={2}>
  <Card title="Kanal-Plugins" href="/de/plugins/sdk-channel-plugins">
    Schritt-für-Schritt-Anleitung, die diese Hilfsfunktionen im Kontext von Kanal-Plugins verwendet.
  </Card>
  <Card title="Provider-Plugins" href="/de/plugins/sdk-provider-plugins">
    Schritt-für-Schritt-Anleitung, die diese Hilfsfunktionen im Kontext von Provider-Plugins verwendet.
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` ist die aktuelle OpenClaw-Produktversion. Sie stammt aus der gemeinsamen Versionsauflösung, sodass Plugins denselben Wert sehen, den die CLI meldet.

## Laden und Schreiben der Konfiguration

Bevorzugen Sie eine Konfiguration, die bereits an den aktiven Aufrufpfad übergeben wurde, beispielsweise `api.config` während der Registrierung oder ein `cfg`-Argument bei Kanal-/Provider-Callbacks. Dadurch fließt ein einziger Prozess-Snapshot durch die Verarbeitung, statt die Konfiguration in häufig ausgeführten Pfaden erneut zu parsen.

Verwenden Sie `api.runtime.config.current()` nur, wenn ein langlebiger Handler den aktuellen Prozess-Snapshot benötigt und dieser Funktion keine Konfiguration übergeben wurde. Der zurückgegebene Wert ist schreibgeschützt; klonen Sie ihn oder verwenden Sie vor der Bearbeitung eine Mutationshilfsfunktion.

Tool-Factories erhalten `ctx.runtimeConfig` sowie `ctx.getRuntimeConfig()`. Verwenden Sie den Getter innerhalb des `execute`-Callbacks eines langlebigen Tools, wenn sich die Konfiguration nach der Erstellung der Tool-Definition ändern kann.

Persistieren Sie Änderungen mit `api.runtime.config.mutateConfigFile(...)` oder `api.runtime.config.replaceConfigFile(...)`. Jeder Schreibvorgang muss eine explizite `afterWrite`-Richtlinie wählen:

- `afterWrite: { mode: "auto" }` überlässt die Entscheidung dem Planer für das Neuladen des Gateway.
- `afterWrite: { mode: "restart", reason: "..." }` erzwingt einen sauberen Neustart, wenn der schreibende Code weiß, dass ein Hot Reload unsicher ist.
- `afterWrite: { mode: "none", reason: "..." }` unterdrückt automatisches Neuladen bzw. Neustarten nur, wenn der Aufrufer für die Folgemaßnahme verantwortlich ist.

Die Mutationshilfsfunktionen geben `afterWrite` sowie eine typisierte `followUp`-Zusammenfassung zurück, damit Aufrufer protokollieren oder testen können, ob sie einen Neustart angefordert haben. Wann dieser Neustart tatsächlich erfolgt, bleibt in der Verantwortung des Gateway.

Verwenden Sie `current()`, ein übergebenes `cfg`, `mutateConfigFile(...)` oder
`replaceConfigFile(...)` für den Zugriff auf die Laufzeitkonfiguration und für Schreibvorgänge.

Bevorzugen Sie bei direkten SDK-Importen die spezifischen Konfigurations-Unterpfade gegenüber dem breiten `openclaw/plugin-sdk/config-runtime`-Kompatibilitäts-Barrel: `config-contracts` für Typen, `runtime-config-snapshot` für aktuelle Prozess-Snapshots und `config-mutation` für Schreibvorgänge. Lesen Sie auf den Eintrag begrenzte Werte aus `api.pluginConfig`; verwenden Sie einen bereitgestellten Tool-Kontext nur für dessen laufzeitweiten Konfigurations-Snapshot und führen Sie Plugin-spezifische Zusammenführungen weiterhin an dieser Grenze durch. Tests gebündelter Plugins sollten diese spezifischen Unterpfade direkt mocken, statt das breite Kompatibilitäts-Barrel zu mocken.

Interner OpenClaw-Laufzeitcode folgt demselben Prinzip: Laden Sie die Konfiguration einmal an der CLI-, Gateway- oder Prozessgrenze und reichen Sie diesen Wert anschließend weiter. Erfolgreiche Mutationsschreibvorgänge aktualisieren den Prozess-Laufzeit-Snapshot und erhöhen dessen interne Revision; langlebige Caches sollten den laufzeiteigenen Cache-Schlüssel verwenden, statt die Konfiguration lokal zu serialisieren. Für langlebige Laufzeitmodule gibt es einen Scanner ohne Toleranz für kontextlose `loadConfig()`-Aufrufe; verwenden Sie ein übergebenes `cfg`, ein Anfrage-`context.getRuntimeConfig()` oder `getRuntimeConfig()` an einer expliziten Prozessgrenze.

Ausführungspfade von Providern und Kanälen müssen den aktiven Snapshot der Laufzeitkonfiguration verwenden, nicht einen Datei-Snapshot, der zum Zurücklesen oder Bearbeiten der Konfiguration zurückgegeben wurde. Datei-Snapshots bewahren Quellwerte wie SecretRef-Markierungen für die Benutzeroberfläche und Schreibvorgänge; Provider-Callbacks benötigen die aufgelöste Laufzeitansicht. Wenn eine Hilfsfunktion entweder mit dem aktiven Quell-Snapshot oder dem aktiven Laufzeit-Snapshot aufgerufen werden kann, leiten Sie ihn vor dem Lesen von Anmeldedaten durch `selectApplicableRuntimeConfig()`.

## Wiederverwendbare Laufzeitdienstprogramme

Verwenden Sie eingehende `botLoopProtection`-Fakten für eingehende, von Bots verfasste Nachrichten. Der Kern wendet vor dem Sitzungsdatensatz und der Weiterleitung die gemeinsame gleitende In-Memory-Fenstersperre an, ohne die Richtlinie an einen einzelnen Kanal zu binden. Die Sperre verfolgt `(scopeId, conversationId, participant pair)`-Schlüssel, zählt beide Richtungen eines Paares gemeinsam, wendet nach Überschreitung des Fensterbudgets eine Abkühlzeit an und entfernt inaktive Einträge bei Gelegenheit.

Kanal-Plugins, die dieses Verhalten für Betreiber verfügbar machen, sollten für Basisbudgets bevorzugt die gemeinsame `channels.defaults.botLoopProtection`-Struktur verwenden und anschließend kanal-/providerspezifische Überschreibungen ergänzen. Die gemeinsame Konfiguration verwendet Sekunden, da sie benutzerseitig sichtbar ist:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

Übergeben Sie normalisierte Bot-Paar-Fakten zusammen mit dem aufgelösten Turn. Der Kern löst Standardwerte, Einheitenumrechnung und die Semantik von `enabled` auf:

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

Verwenden Sie `openclaw/plugin-sdk/pair-loop-guard-runtime` nur direkt für benutzerdefinierte
Ereignisschleifen zwischen zwei Parteien, die nicht durch den gemeinsamen Runner für eingehende Antworten laufen.

## Laufzeit-Namespaces

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    Agentenidentität, Verzeichnisse und Sitzungsverwaltung.

    ```typescript
    // Arbeitsverzeichnis des Agenten auflösen (agentId ist erforderlich)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // Agenten-Workspace auflösen
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // Agentenidentität abrufen
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // Standard-Denkniveau abrufen
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // Ein benutzerseitig angegebenes Denkniveau anhand des aktiven Provider-Profils validieren
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // Niveau an einen eingebetteten Lauf übergeben
    }

    // Zeitüberschreitung des Agenten abrufen
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // Sicherstellen, dass der Workspace vorhanden ist
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // Einen eingebetteten Agenten-Turn ausführen
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "Fassen Sie die neuesten Änderungen zusammen",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` ist die neutrale Hilfsfunktion zum Starten eines normalen OpenClaw-Agenten-Turns aus Plugin-Code. Sie verwendet dieselbe Provider-/Modellauflösung und Auswahl des Agenten-Harness wie durch Kanäle ausgelöste Antworten.

    `runEmbeddedPiAgent(...)` bleibt als veralteter Kompatibilitätsalias für vorhandene Plugins bestehen. Neuer Code sollte `runEmbeddedAgent(...)` verwenden.

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` stellt Aufrufern, die eingebettete Läufe für `cliBackendDispatch: "subscription-auth"` aktivieren, die Entscheidung des eingebetteten Runners über die Weiterleitung an das CLI-Backend zur Verfügung (Route, die deklarierte `subscriptionAuthDispatch`-Fähigkeit des Backends, gespeicherter Anmeldedatenmodus – unter Berücksichtigung eines explizit festgelegten `authProfileId`). Die Funktion gibt `{ provider }` zurück, wenn der Lauf über das CLI-Backend ausgeführt würde, und `undefined`, wenn er beim direkten Durchreichen bleibt, sodass Aufrufer Zeitüberschreitungen für den Lauf bemessen können, der tatsächlich ausgeführt wird.

    `resolveThinkingPolicy(...)` gibt die vom Provider/Modell unterstützten Denkniveaus und den optionalen Standardwert zurück. Provider-Plugins verwalten das modellspezifische Profil über ihre Thinking-Hooks. Daher sollten Tool-Plugins diese Laufzeithilfsfunktion aufrufen, statt Provider-Listen zu importieren oder zu duplizieren.

    `normalizeThinkingLevel(...)` konvertiert Benutzereingaben wie `on`, `x-high` oder `extra high` in das kanonisch gespeicherte Niveau, bevor es anhand der aufgelösten Richtlinie geprüft wird.

    **Hilfsfunktionen für den Sitzungsspeicher** befinden sich unter `api.runtime.agent.session`:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // Sitzungszeilen durchlaufen, ohne von der alten sessions.json-Struktur abhängig zu sein.
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
        // Sitzung erstellen oder aktualisieren und anschließend signal an den zugelassenen Agentenlauf übergeben.
      },
    );
    ```

    Bevorzugen Sie `getSessionEntry(...)`, `listSessionEntries(...)`, `patchSessionEntry(...)` oder `upsertSessionEntry(...)` für Sitzungsabläufe. Diese Hilfsfunktionen adressieren Sitzungen anhand der Agenten-/Sitzungsidentität, sodass Plugins nicht von der alten `sessions.json`-Speicherstruktur abhängen. Verwenden Sie `preserveActivity: true` für reine Metadaten-Patches, die die Sitzungsaktivität nicht aktualisieren sollen, und `replaceEntry: true` nur, wenn der Callback einen vollständigen Eintrag zurückgibt und gelöschte Felder gelöscht bleiben müssen. Doctor- und Migrationspfade können `fallbackEntry`, `skipMaintenance` und `requireWriteSuccess` für eine einzelne atomare Reparatur des kanonischen Speichers kombinieren.

    `createSessionEntry(...)` erstellt eine neue kanonische Sitzungszeile und ein Transkript. Die vertrauenswürdige `initialEntry`-Oberfläche ist bewusst eng gefasst: ein nicht leeres `agentHarnessId`, optionales `modelSelectionLocked: true` und optionales `pluginExtensions`. Die injizierte Laufzeit akzeptiert über `registerAgentHarness(...)` ausschließlich Harness-IDs, die dem aufrufenden Plugin gehören; dies ist eine Eigentumsinvariante und keine Sandbox zwischen prozessinternen Plugins. Eine bereits vorhandene Zeile wird abgelehnt; `label` und `spawnedCwd` sind separate Erstellungsfelder und keine Patches vertrauenswürdiger Einträge.

    Die Erstellung hält den Mutationszaun des Sitzungslebenszyklus über `afterCreate`, sodass neue Arbeit wartet, bis die Plugin-eigene Initialisierung abgeschlossen ist, während bereits zuvor zugelassene Arbeit zum Fehlschlagen der Erstellung führt. Der Callback erhält einen Klon des erstellten Zustands. Wenn er einen Patch zurückgibt, darf dieser Patch ausschließlich `pluginExtensions` enthalten; sein Wert ist das vollständige endgültige Feld `pluginExtensions`. Ein Fehler des Callbacks oder der abschließenden Persistierung setzt die unveränderte neue Zeile und das Transkript zurück; ein geschütztes Rollback bewahrt eine parallel geänderte oder beanspruchte Zeile. `recoverMatchingInitialEntry: true` dient ausschließlich dazu, eine unterbrochene Initialisierung erneut zu versuchen, wenn die persistierten vertrauenswürdigen Felder exakt übereinstimmen; die Wiederherstellung erfordert, dass `afterCreate` einen abschließenden Patch zurückgibt.

    Verwenden Sie `runWithWorkAdmission(...)`, wenn ein Plugin Arbeit an einer persistierten Sitzung startet. Der Callback lehnt archivierte oder parallel ersetzte Sitzungen ab, koordiniert Archivierungs-, Zurücksetzungs- und Löschmutationen bis zum Abschluss und erhält ein `AbortSignal`, das an den Agentenlauf weitergegeben werden muss. Ein Harness kann über sein experimentelles Registrierungsfeld `delegatedExecutionPluginIds` explizit vertrauenswürdige Ausführungsdelegierte benennen. Delegierte können ausschließlich eine exakt vorhandene, modellgesperrte Sitzung zulassen und ausführen; alle Sitzungsmutationen bleiben auf den Eigentümer des Harness beschränkt. Siehe [Agenten-Harness-Plugins](/de/plugins/sdk-agent-harness#delegated-execution).

    Wartungs- und Reparatur-Plugins können `deleteSessionEntry(...)` für einen einzelnen sitzungsbezogenen Eintrag, `cleanupSessionLifecycleArtifacts(...)` für vom Lebenszyklus verwaltete temporäre Sitzungen und `resolveSessionStoreBackupPaths(...)` vor der Änderung eines Speichers verwenden. Übergeben Sie `expectedSessionId` und `expectedUpdatedAt`, wenn eine Löschung nicht mit einer gleichzeitigen Sitzungsaktualisierung konkurrieren darf; verwenden Sie `expectedSessionId: null`, wenn der frühere Snapshot keine Sitzungs-ID enthielt. Diese Hilfsfunktionen sind eng begrenzte Reparatur-/Lebenszyklus-Schnittstellen und keine allgemeine API zum Löschen von Speichern.

    `resolveStorePath(...)` und `updateSessionStoreEntry(...)` vervollständigen die Sitzungshilfsfunktionen: `resolveStorePath` ermittelt den Pfad des Sitzungsspeichers für einen bestimmten Geltungsbereich, und `updateSessionStoreEntry({ storePath, sessionKey, update })` aktualisiert einen Eintrag direkt anhand des Speicherpfads, wenn dieser dem Aufrufer bereits bekannt ist.

    `loadTranscriptEventsSync(...)` steht für synchrone Doctor- und Reparaturpfade zur Verfügung, die die asynchrone Transkript-Laufzeit nicht verwenden können. Die Funktion gibt rohe `SessionStoreTranscriptEvent`-Datensätze zurück. Normaler Plugin-Laufzeitcode sollte `openclaw/plugin-sdk/session-transcript-runtime` bevorzugen.

    `formatSqliteSessionFileMarker(...)`, `parseSqliteSessionFileMarker(...)` und `sqliteSessionFileMarkerMatchesSession(...)` sind Übergangshilfsfunktionen für Code, der noch ein Legacy-Feld namens `sessionFile` empfängt. Eine geparste SQLite-Markierung bezeichnet ein aktives SQLite-Transkriptziel; sie ist kein Dateisystempfad. Neue APIs sollten statt Markierungszeichenfolgen eine typisierte Sitzungsidentität übermitteln.

    Importieren Sie für das Lesen und Schreiben von Transkripten `openclaw/plugin-sdk/session-transcript-runtime` und verwenden Sie `resolveSessionTranscriptIdentity(...)`, `resolveSessionTranscriptTarget(...)`, `readSessionTranscriptEvents(...)`, `readSessionTranscriptRawDelta(...)`, `readSessionTranscriptVisibleMessageDelta(...)`, `readVisibleSessionTranscriptMessageEntries(...)`, `appendSessionTranscriptMessageByIdentity(...)`, `publishSessionTranscriptUpdateByIdentity(...)` oder `withSessionTranscriptWriteLock(...)` mit `{ agentId, sessionKey, sessionId }`. Mit diesen APIs können Plugins ein Transkript identifizieren, rohe Ereignisse oder sichtbare, verzweigungssichere Nachrichteneinträge lesen, Nachrichten anhängen, Aktualisierungen veröffentlichen und zugehörige Vorgänge unter derselben Transkript-Schreibsperre ausführen, ohne von Pfaden aktiver Transkriptdateien abhängig zu sein. `readVisibleSessionTranscriptMessageEntries(...)` gibt geordnete Lesemetadaten zurück; das Feld `seq` ist kein fortsetzbarer Cursor.

    `appendSessionTranscriptMessageByIdentity(...)` ist eine Low-Level-Funktion zum Anhängen einer bereits kanonischen Nachricht. Plugins dürfen keine medienhaltigen Benutzerzeilen mit `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType` oder `MediaTypes` auf oberster Ebene erzeugen. Der Kanaleingang sollte geordnete Fakten über `MsgContext.media` übergeben und die Persistierung des Benutzerzugs dem Host überlassen. Eine vom Host vorbereitete persistierte Benutzernachricht enthält kanonische geordnete Fakten unter `message.__openclaw.media`; die generische API zum Anhängen leitet keine parallelen Legacy-Arrays ab und repariert sie auch nicht.

    `readSessionTranscriptRawDelta(...)` gibt ein begrenztes Ergebnis vom Typ `page`, `reset` oder `missing` zurück. Übergeben Sie den opaken `page.cursor` beim nächsten Aufruf. Reine Anhängevorgänge behalten den Cursor bei, während das Ersetzen des Transkripts `reset` mit einem neuen Bootstrap-Cursor zurückgibt. Seiten umfassen standardmäßig 1.000 Ereignisse und 1.000.000 serialisierte Bytes; Aufrufer können bis zu 10.000 Ereignisse und 64 MiB anfordern. Wenn bereits das nächste Ereignis `maxBytes` überschreitet, ist die Seite leer und meldet `requiredBytes`; versuchen Sie es erneut mit mindestens diesem Byte-Limit, sofern es nicht größer als 64 MiB ist. Größere Einzelereignisse erfordern die API zum vollständigen Lesen. Ein Cursor kennzeichnet ausschließlich eine Position und gewährt niemals Zugriff auf eine andere Sitzung.

    `readSessionTranscriptVisibleMessageDelta(...)` stellt dieselbe begrenzte Bootstrap-und-Fortsetzungsstruktur für die hostverwaltete Projektion aktiver Nachrichten bereit. Die Funktion gibt Nachrichten von der ältesten bis zur neuesten zurück, sodass Kontext-Engines den anfänglichen Verlauf vollständig abarbeiten und den opaken Cursor als ihre Fortschrittsmarke persistieren können. Speichern Sie den Cursor unverändert und geben Sie ihn unverändert zurück; er ist ein Fortsetzungshinweis und kein Autorisierungsnachweis. Lineare Anhängevorgänge werden nach der zuletzt zurückgegebenen Nachricht fortgesetzt. Das Ersetzen des Transkripts, ein Cursor, dessen Anker den aktiven Zweig verlassen hat oder innerhalb dieses Zweigs verschoben wurde, fehlerhafte Cursor und sitzungsübergreifende Cursor geben `reset` mit einem neuen Bootstrap-Cursor zurück. Die Standardwerte und Obergrenzen für Anzahl und Bytes entsprechen der Rohdaten-Delta-API. Während die aktive Projektion nach einer Zweigänderung neu aufgebaut wird, lautet das Ergebnis `unavailable` mit dem Grund `projection_rebuilding`; versuchen Sie es später erneut, statt auf eine aktive Transkriptdatei zurückzugreifen.

    Die Legacy-Hilfsfunktionen für den gesamten Speicher und aktive Transkriptdateien werden nicht mehr aus dem Plugin-SDK exportiert. Verwenden Sie die auf Einträge begrenzten Hilfsfunktionen für Sitzungsmetadaten und die Hilfsfunktionen zur Transkriptidentität für Vorgänge mit aktiven Transkripten. Archivierungs-/Support-Workflows, die Dateiartefakte benötigen, sollten ihre dedizierten Archivschnittstellen statt der Laufzeit-APIs aktiver Sitzungen verwenden.

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    Konstanten für Standardmodell und -Provider:

    ```typescript
    const model = api.runtime.agent.defaults.model; // z. B. "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // z. B. "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    Führen Sie eine hostverwaltete Textvervollständigung aus, ohne interne Provider-Komponenten zu importieren oder
    die Vorbereitung von OpenClaw für Modell, Authentifizierung und Basis-URL zu duplizieren.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "Fassen Sie dieses Transkript zusammen." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    Die Provider-Orchestrierung kann außerdem den konfigurierten Lebenszyklus des lokalen Dienstes
    übernehmen, bevor eine HTTP-Anfrage gestellt wird:

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
      // Senden Sie die Provider-Anfrage und verarbeiten Sie sie vollständig.
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` ist ein stabiler, generischer SDK-Vertrag für Provider-Dienste.
    Der Host ermittelt die Prozesskonfiguration aus
    `models.providers.<providerId>.localService`; Aufrufer können weder einen
    Befehl noch Argumente, eine Umgebung oder eine Lebenszyklusrichtlinie angeben. Prozesserzeugung,
    Bereitschaft, Diagnose und die Richtlinie zum Beenden bei Inaktivität bleiben hostintern.

    Übergeben Sie die exakte konfigurierte Provider-ID und die aufgelöste Basis-URL der Anfrage. Ersetzen Sie
    Aliasse nicht durch eine Adapter-ID: Separate Aliasse können auf separate
    lokale GPU-Hosts verweisen. Der Host weist Endpunkte zurück, die nicht mit der konfigurierten
    Basis-URL des Providers übereinstimmen, abgesehen von der von Ollama- und LM-
    Studio-Adaptern verwendeten Normalisierung `/v1`. Der Host verwaltet die Serialisierung des Starts, Bereitschaftsprüfungen,
    Anfrage-Leases, Abbruchbehandlung und das Herunterfahren bei Inaktivität.

    Die Hilfsfunktion verwendet denselben Vorbereitungspfad für einfache Vervollständigungen wie die integrierte
    OpenClaw-Laufzeit sowie den hostverwalteten Snapshot der Laufzeitkonfiguration. Kontext-Engines
    erhalten eine sitzungsgebundene `llm.complete`-Fähigkeit, sodass Modellaufrufe den Agenten
    der aktiven Sitzung verwenden und nicht stillschweigend auf den Standardagenten zurückfallen. Das
    Ergebnis enthält die Zuordnung zu Provider, Modell und Agent sowie normalisierte Token-,
    Cache- und geschätzte Kostennutzung, sofern verfügbar.

    Setzen Sie `reasoning`, um einen Reasoning-Aufwand für das ausgewählte Modell anzufordern. Der
    Host normalisiert vor dem Senden der Vervollständigung die kanonischen Denkstufen (`off`, `minimal`, `low`,
    `medium`, `high`, `xhigh`, `adaptive`, `max` und `ultra`) für den ausgewählten
    Provider und das Modell. `adaptive` wird zu
    `medium`; `max` und `ultra` werden, sofern unterstützt, zu `max`, andernfalls zu `xhigh`.

    <Warning>
    Modellüberschreibungen erfordern die Zustimmung des Betreibers über `plugins.entries.<id>.llm.allowModelOverride: true` in der Konfiguration. Verwenden Sie `plugins.entries.<id>.llm.allowedModels`, um vertrauenswürdige Plugins auf bestimmte kanonische `provider/model`-Ziele zu beschränken. Agentenübergreifende Vervollständigungen erfordern `plugins.entries.<id>.llm.allowAgentIdOverride: true`.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    Rufen Sie prozessintern eine andere Gateway-Methode auf und bewahren Sie dabei die vertrauenswürdige Laufzeitidentität
    des aktuellen Plugins. Dies ist für gebündelte oder vertrauenswürdige offizielle Plugins vorgesehen, die Plugin-eigene
    Gateway-Fähigkeiten kombinieren, ohne eine Loopback-WebSocket-Verbindung zu öffnen.

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    Anfragen verwenden den Geltungsbereich `operator.write` und gewähren keinen Administrator-Geltungsbereich. Aufrufe beliebiger externer
    Plugins werden abgelehnt. Fehlgeschlagene Methoden lösen einen `GatewayClientRequestError` aus und bewahren strukturierte
    `details`-, Wiederholungsmetadaten und den Gateway-Fehlercode für Wiederherstellungsabläufe. Verwenden Sie `isAvailable()`,
    bevor Sie diesen Pfad aus Tools auswählen, die auch in eigenständigen Agentenprozessen ausgeführt werden können.

  </Accordion>
  <Accordion title="api.runtime.subagent">
    Starten und verwalten Sie im Hintergrund ausgeführte Subagentenläufe.

    ```typescript
    // Starten Sie einen Subagentenlauf
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "Erweitern Sie diese Abfrage zu gezielten weiterführenden Suchen.",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // optionale Überschreibung
      model: "gpt-5.6-sol", // optionale Überschreibung
      deliver: false,
    });

    // Auf Abschluss warten
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // Sitzungsnachrichten lesen
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // Eine Sitzung löschen
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    Modellüberschreibungen (`provider`/`model`) erfordern die Zustimmung des Betreibers über `plugins.entries.<id>.subagent.allowModelOverride: true` in der Konfiguration. Nicht vertrauenswürdige Plugins können weiterhin Subagenten ausführen, Überschreibungsanfragen werden jedoch abgelehnt.
    </Warning>

    `toolsAlsoAllow` fügt der normalen Tool-Oberfläche des Workers exakt benannte, eindeutig dem aufrufenden Plugin zugeordnete Tools hinzu. Die Laufzeit lehnt Core-Tools sowie Namen ab, die mit einem anderen Plugin geteilt werden. Profile und Tool-Richtlinien des Betreibers gelten weiterhin, einschließlich expliziter Zulassungs- und Sperrlisten.

    `deleteSession(...)` kann Sitzungen löschen, die dasselbe Plugin über `api.runtime.subagent.run(...)` erstellt hat. Das Löschen beliebiger Benutzer- oder Betreibersitzungen erfordert weiterhin eine Gateway-Anfrage mit Administrator-Geltungsbereich.

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    Prüfen Sie die effektive Sandbox-Arbeitsbereichsberechtigung für eine Agentensitzung.

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

    Das Ergebnis gibt an, ob diese Sitzung in einer Sandbox ausgeführt wird, ob ihr Arbeitsbereich
    nicht verfügbar, schreibgeschützt oder beschreibbar ist, sowie optional `confinementError`,
    wenn die effektive Docker-, Tool-, Sitzungs-, Browser- oder erhöhte Richtlinie
    diesen Arbeitsbereich verlassen kann. Verwenden Sie dies für hostverwaltete Delegierungsentscheidungen, die
    einem Worker nicht mehr Berechtigungen gewähren dürfen als seinem Aufrufer. Es handelt sich um eine Hilfsfunktion
    zur Bestätigung, nicht um einen Ersatz für die Prüfung der eigenen Autorisierung des Aufrufers.

    `prepareWorkspaceAuthority(...)` führt dieselbe Richtlinienprüfung durch und
    bereitet zusätzlich die Docker-Sandbox für `workspaceDir` vor. Die Funktion lehnt einen aktiven Container ab,
    dessen Hash der laufenden Konfiguration nicht mit den angeforderten Einbindungen oder der Richtlinie übereinstimmt. Übergeben Sie
    ausschließlich exakte Tool-Namen, deren registrierte Implementierungen das aufrufende Plugin
    einschränkt; Platzhalterpräfixe belegen keine Tool-Zuordnung.

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Listen Sie verbundene Nodes auf und rufen Sie einen auf einem Node gehosteten Befehl aus vom Gateway geladenem Plugin-Code oder aus Plugin-CLI-Befehlen auf. Verwenden Sie dies, wenn ein Plugin lokale Arbeit auf einem gekoppelten Gerät verwaltet, beispielsweise eine Browser- oder Audio-Bridge auf einem anderen Mac.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` enthält die von jedem verbundenen Node angekündigten
    `nodePluginTools`-Deskriptoren, wenn dieser Node Plugin- oder MCP-gestützte
    Tools für den Agent bereitstellt. Diese Deskriptoren bilden den aktuellen Verbindungsstatus ab: Der Gateway
    verwirft sie, wenn der Node die Verbindung trennt, und ein Node kann sie nach Änderungen am lokalen Plugin-/MCP-Bestand
    durch `node.pluginTools.update` ersetzen.

    Innerhalb des Gateway läuft diese Runtime prozessintern. In Plugin-CLI-Befehlen ruft sie den konfigurierten Gateway über RPC auf, sodass Befehle wie `openclaw googlemeet recover-tab` gekoppelte Nodes über das Terminal prüfen können. Node-Befehle durchlaufen weiterhin die normale Node-Kopplung des Gateway, Befehls-Positivlisten, Richtlinien für Plugin-Node-Aufrufe und die lokale Befehlsverarbeitung des Node.

    Plugins, die auf Nodes gehostete Agent-Tools bereitstellen, können `agentTool.defaultPlatforms` für ungefährliche Befehle festlegen, die standardmäßig in die Positivliste aufgenommen werden sollen. Lassen Sie die Option weg, wenn Betreiber die Befehle explizit über `gateway.nodes.commands.allow` aktivieren müssen. Gefährliche auf Nodes gehostete Befehle sollten mit `api.registerNodeInvokePolicy(...)` eine Richtlinie für Node-Aufrufe registrieren; die Richtlinie wird im Gateway nach der Prüfung der Befehls-Positivliste und vor der Weiterleitung des Befehls an den Node ausgeführt, sodass direkte `node.invoke`-Aufrufe, auf Nodes gehostete Plugin-Tools und übergeordnete Plugin-Tools denselben Durchsetzungspfad verwenden.

    <Warning>
    Das optionale Feld `scopes` fordert Gateway-Betreiberbereiche für den Aufruf an. OpenClaw berücksichtigt es nur für gebündelte Plugins und vertrauenswürdige Installationen offizieller Plugins; Anforderungen anderer Plugins erweitern die Berechtigungen des Aufrufs nicht. Verwenden Sie es nur, wenn ein vertrauenswürdiges Plugin einen Node-Befehl mit einem strengeren Gateway-Bereich wie `operator.admin` aufrufen muss.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Bindet den Zustand von Task Flow und Task Run an einen vorhandenen OpenClaw-Sitzungsschlüssel oder einen vertrauenswürdigen Tool-Kontext.

    - `api.runtime.tasks.managedFlows` unterstützt Änderungen: Task Flows erstellen, fortsetzen und abbrechen.
    - `api.runtime.tasks.flows` und `api.runtime.tasks.runs` sind schreibgeschützte DTO-Ansichten für Auflistungen und Statusabfragen; beide stellen `bindSession(...)` / `fromToolContext(...)` sowie `get`, `list`, `findLatest` und `resolve` bereit.

    Task Flow verfolgt den dauerhaften Zustand mehrstufiger Workflows. Es ist kein Planer:
    Verwenden Sie Cron oder `api.session.workflow.scheduleSessionTurn(...)` für zukünftige
    Aktivierungen und anschließend `managedFlows` während des geplanten Durchlaufs, wenn diese Arbeit
    Ablaufzustand, untergeordnete Aufgaben, Wartephasen oder Abbruchmöglichkeiten benötigt.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Neue Pull Requests prüfen",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "PR #123 prüfen",
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

    Verwenden Sie `bindSession({ sessionKey, requesterOrigin })`, wenn Sie bereits über einen vertrauenswürdigen OpenClaw-Sitzungsschlüssel aus Ihrer eigenen Bindungsschicht verfügen. Stellen Sie keine Bindung anhand ungeprüfter Benutzereingaben her.

  </Accordion>
  <Accordion title="api.runtime.tts">
    Text-zu-Sprache-Synthese.

    ```typescript
    // Standard-TTS
    const clip = await api.runtime.tts.textToSpeech({
      text: "Hallo von OpenClaw",
      cfg: api.config,
    });

    // Für Telefonie optimiertes TTS
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "Hallo von OpenClaw",
      cfg: api.config,
    });

    // Verfügbare Stimmen auflisten
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    Verwendet die zentrale `tts`-Konfiguration und Provider-Auswahl. Gibt einen PCM-Audiopuffer und die Abtastrate zurück. `textToSpeechStream` ist ebenfalls für die Streaming-Synthese verfügbar.

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    Bild-, Audio- und Videoanalyse.

    ```typescript
    // Ein Bild beschreiben
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // Audio transkribieren
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // optional, wenn MIME nicht abgeleitet werden kann
    });

    // Ein Video beschreiben
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // Allgemeine Dateianalyse
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // Strukturierte Bildextraktion über einen bestimmten Provider/ein bestimmtes Modell.
    // Mindestens ein Bild einfügen; Texteingaben dienen als ergänzender Kontext.
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
        { type: "text", text: "Bevorzuge den gedruckten Gesamtbetrag gegenüber handschriftlichen Notizen." },
      ],
      instructions: "Anbieter, Gesamtbetrag und durchsuchbare Tags extrahieren.",
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

    Gibt `{ text: undefined }` zurück, wenn keine Ausgabe erzeugt wird (z. B. bei übersprungener Eingabe).

    `describeImageFileWithModel(...)` beschreibt ein bereits bekanntes Bild über einen bestimmten Provider/ein bestimmtes Modell und umgeht dabei die standardmäßige Auflösung des aktiven Modells, die `describeImageFile(...)` verwendet.

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    Bilderzeugung.

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "Ein Roboter malt einen Sonnenuntergang",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    Videoerzeugung, entsprechend der Struktur der Bilderzeugung.

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "Eine Drohnenaufnahme, die bei Sonnenaufgang über eine Küste fliegt",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    Musikerzeugung, entsprechend der Struktur der Bilderzeugung.

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "Ein schwungvoller Lo-Fi-Titel für eine Programmiersitzung",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Websuche.

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "OpenClaw Plugin SDK", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    Medien-Dienstprogramme auf niedriger Ebene.

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
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
    Aktueller Snapshot der Runtime-Konfiguration und transaktionale Konfigurationsschreibvorgänge. Bevorzugen Sie
    eine Konfiguration, die bereits an den aktiven Aufrufpfad übergeben wurde; verwenden Sie
    `current()` nur, wenn der Handler den Prozess-Snapshot direkt benötigt.

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` und `replaceConfigFile(...)` geben einen `followUp`-Wert zurück,
    beispielsweise `{ mode: "restart", requiresRestart: true, reason }`,
    der die Absicht des Schreibers erfasst, ohne dem
    Gateway die Kontrolle über den Neustart zu entziehen.

  </Accordion>
  <Accordion title="api.runtime.system">
    Dienstprogramme auf Systemebene.

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // Veralteter Kompatibilitätsalias.
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` führt sofort einen einzelnen Heartbeat-Zyklus aus und umgeht dabei den normalen Zusammenführungstimer. Übergeben Sie `{ heartbeat: { target: "last" } }`, um die Zustellung an den zuletzt aktiven Kanal statt der standardmäßigen `target: "none"`-Unterdrückung zu erzwingen.

    `runCommandWithTimeout(...)` gibt die erfassten Werte `stdout` und `stderr`, optionale
    Kürzungsanzahlen, `code`, `signal`, `killed`, `termination` und
    `noOutputTimedOut` zurück. Ergebnisse bei Zeitüberschreitung und Zeitüberschreitung ohne Ausgabe melden `code: 124`,
    wenn der untergeordnete Prozess keinen von null abweichenden Exit-Code liefert. Signalbedingte Beendigungen ohne Zeitüberschreitung
    können dennoch `code: null` zurückgeben; verwenden Sie daher `termination` und
    `noOutputTimedOut`, um die Gründe für Zeitüberschreitungen zu unterscheiden.

  </Accordion>
  <Accordion title="api.runtime.events">
    Ereignisabonnements.

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
    Protokollierung.

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    Auflösung der Modell- und Provider-Authentifizierung.

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // Für Anfragen bereite Authentifizierung, einschließlich Provider-Runtime-Austauschvorgängen (z. B. OAuth-Aktualisierung)
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    Auflösung des Zustandsverzeichnisses und SQLite-gestützter schlüsselbasierter Speicher.

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
      new TextEncoder().encode("binary or text payload"),
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

    Schlüsselbasierte Speicher überstehen Neustarts und werden anhand der Runtime-gebundenen Plugin-ID isoliert. Verwenden Sie `registerIfAbsent(...)` für atomare Deduplizierungsansprüche: Es gibt `true` zurück, wenn der Schlüssel fehlte oder abgelaufen war und registriert wurde, oder `false`, wenn bereits ein aktiver Wert vorhanden ist, ohne dessen Wert, Erstellungszeit oder TTL zu überschreiben. Verwenden Sie `deleteIf(...)`, wenn die Bereinigung nur den zuvor beobachteten Wert entfernen darf; das synchrone Prädikat und die Löschung werden in einer SQLite-Transaktion ausgeführt. Grenzwerte: `maxEntries` pro Namespace, 50,000 aktive Zeilen pro Plugin, JSON-Werte unter 64KB und optionaler TTL-Ablauf. Standardmäßig entfernt ein Schreibvorgang beim Erreichen eines der beiden Zeilenlimits die ältesten aktiven Zeilen aus dem gerade beschriebenen Namespace; gleichgeordnete Namespaces werden für diesen Schreibvorgang nicht geräumt, und der Schreibvorgang schlägt weiterhin fehl, wenn der Namespace nicht genügend Zeilen freigeben kann. Legen Sie `overflowPolicy: "reject-new"` für dauerhafte Eigentümerschaftsdatensätze fest, die niemals geräumt werden dürfen: Neue Schlüssel schlagen bei beiden Limits fehl, während vorhandene Schlüssel weiterhin aktualisiert werden können.

    `openSyncKeyedStore<T>(...)` gibt dieselbe Speicherstruktur mit synchronen Methoden zurück (`register`, `registerIfAbsent`, `deleteIf`, `lookup`, `consume`, `clear` geben alle Werte direkt statt Promises zurück), für Aufrufer, die nicht warten können.

    `openBlobStore<TMetadata>(...)` speichert begrenzte binäre Nutzdaten in gemeinsam genutztem SQLite ohne Base64 oder Datei-Sidecars. Es erfordert Byte- und Zeilenlimits pro Eintrag und pro Namespace, kopiert Byte-Arrays an der API-Grenze und listet Metadaten auf, ohne jeden BLOB zu laden. `register(...)` ist ein explizites Upsert, auch für abgelaufene Schlüssel. `registerIfAbsent(...)` ermöglicht kollisionssichere Erstellung: Ein abgelaufener Schlüssel bleibt belegt, bis sein Eigentümer ihn mit `deleteExpiredKey(key)` oder `deleteExpired()` beansprucht. Dadurch bleiben die Metadaten erhalten, die zum Entfernen zugehöriger benannter Artefakte nach dem SQLite-Commit erforderlich sind. Jede Zeile mit einer TTL ist temporär und wird selbst vor ihrem Ablauf von Sicherung und Wiederherstellung ausgeschlossen; lassen Sie die TTL für dauerhaften, wiederherstellbaren Zustand weg. Host-Sicherungen begrenzen jeden BLOB auf 100 MiB, jedes Plugin auf 512 MiB physisch gespeicherter BLOBs und jedes Plugin auf 50,000 physisch gespeicherte Zeilen, einschließlich abgelaufener Zeilen, die auf die Bereinigung durch den Eigentümer warten. Verwenden Sie `registerIfAbsent(...)` mit `overflowPolicy: "reject-new"`, wenn externe Materialisierungen durch Ersetzung oder Räumung nicht unbemerkt verwaisen dürfen.

    `openChannelIngressQueue<TPayload>(...)` öffnet eine persistierte Eingangswarteschlange mit Gültigkeitsbereich für das aufrufende Plugin, um eingehende Ereignisse zu puffern, die über Neustarts hinweg mindestens einmal verarbeitet werden müssen. Wenn die Wiederherstellung veralteter Ansprüche `shouldRecover` verwendet, geben Sie außerdem `shouldRecoverCorrupt` an, falls beschädigte beanspruchte Nutzdaten unter Quarantäne gestellt werden sollen: Die von den Nutzdaten unabhängige Anspruchsidentität ermöglicht es dem Plugin, aktive Eigentümer- und Lane-Richtlinien beizubehalten, bevor die Warteschlange die Zeile mit einem Tombstone versieht.

    `withLease(...)` serialisiert kooperative Plugin-Arbeit über OpenClaw-Prozesse hinweg. Wählen Sie `database: { scope: "shared" }` für einen globalen Eigentümer oder `{ scope: "agent", agentId }` für unabhängige Eigentümerschaft pro Agent. Leiten Sie das `AbortSignal` des Callbacks an jeden Vorgang weiter, der fehlschlagen kann. `assertOwned()` ist ein Zeitpunkt-Checkpoint vor dem Start eines weiteren wichtigen Schritts; der Host überprüft die Eigentümerschaft außerdem nach dem Callback. Der Verlust der Lease oder ein Abbruch durch den Aufrufer bricht das Signal ab. Warten auf den Erwerb und Heartbeats erfolgen außerhalb kurzer synchroner SQLite-Transaktionen; Plugins erhalten niemals Datenbankpfade oder Handles. Dies ist eine kooperative Abbruchfunktion, kein Fencing-Token und keine Autorisierung für externe Schreibvorgänge ohne Fencing.

    `openChannelIngressDrain(...)` öffnet den zentralen kanalunabhängigen Worker über dieser Warteschlange (oder erstellt eine Warteschlange, wenn keine angegeben ist). Der Drain verwaltet die Wiederherstellung veralteter Ansprüche, die Serialisierung von Ansprüchen pro Lane, den Abschluss bei Übernahme oder bei Rückkehr der Verteilung, die Wiederholungs-/Dead-Letter-Disposition, die optionale Ablösung vor der Übernahme und das Zeitlimit für einen Stillstand zwischen Anspruch und Übernahme. Binden Sie die Anspruchseigentümerschaft mit `turnAdoptionLifecycle` in die Antwortgenerierung ein (über `bindIngressLifecycleToReplyOptions` aus `plugin-sdk/channel-outbound`). Kanal-Plugins behalten die annahmeseitige Einreihung, die Lane-Ableitung, die Klassifizierung als nicht wiederholbar und sämtliche Autorisierungsrichtlinien für Ablösungen.

    <Warning>
    `openBlobStore`, `openKeyedStore`, `openSyncKeyedStore`, `withLease`, `openChannelIngressQueue` und `openChannelIngressDrain` sind in dieser Version nur für gebündelte Plugins und vertrauenswürdige Installationen offizieller Plugins verfügbar.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    Kanalspezifische Runtime-Hilfsfunktionen (verfügbar, wenn ein Kanal-Plugin geladen ist). Nach Zuständigkeitsbereich gruppiert:

    | Gruppe | Zweck |
    | --- | --- |
    | `text` | Aufteilung (`chunkText`, `chunkMarkdownText`, `resolveChunkMode`), Erkennung von Steuerbefehlen, Konvertierung von Markdown-Tabellen. |
    | `reply` | Verteilung gepufferter Blockantworten, Umschlagformatierung, Auflösung der effektiven Nachrichten-/Verzögerungskonfiguration für Menschen. |
    | `routing` | `buildAgentSessionKey`, `resolveAgentRoute`. |
    | `pairing` | `buildPairingReply`, Lesen/Entfernen aus Zulassungslisten, Upserts von Kopplungsanfragen und aus Anfragen abgeleitete Genehmigungseinträge. |
    | `media` | Herunterladen/Speichern entfernter Medien (siehe unten). |
    | `activity` | Letzte Kanalaktivität erfassen/lesen. |
    | `session` | Sitzungsmetadaten aus eingehenden Ereignissen, Aktualisierungen der letzten Route. |
    | `mentions` | Hilfsfunktionen für Erwähnungsrichtlinien (siehe unten). |
    | `reactions` | Handles für Bestätigungsreaktionen als Anzeigen laufender Verarbeitung. |
    | `groups` | Auflösung von Gruppenrichtlinie und Erwähnungspflicht. |
    | `debounce` | Entprellung eingehender Nachrichten. |
    | `commands` | Befehlsautorisierung und Sperrung von Textbefehlen. |
    | `outbound` | Ausgehenden Adapter eines Kanals laden. |
    | `inbound` | Kontext eingehender Ereignisse erstellen und den gemeinsamen Kernel für eingehende Ereignisse/Antworten ausführen. |
    | `threadBindings` | Leerlaufzeitlimit/maximales Alter für gebundene Sitzungsthreads anpassen. |
    | `runtimeContexts` | Prozesslokalen Kontext pro Kanal/Konto/Fähigkeit registrieren, lesen und überwachen. |

    `api.runtime.channel.media` ist die bevorzugte Schnittstelle für das Herunterladen und Speichern von Kanalmedien:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    Verwenden Sie `saveRemoteMedia(...)`, wenn eine entfernte URL zu einem OpenClaw-Medium werden soll. Verwenden Sie `saveResponseMedia(...)`, wenn das Plugin bereits ein `Response` mit Plugin-eigener Authentifizierung, Weiterleitungs- oder Zulassungslistenverarbeitung abgerufen hat. Verwenden Sie `readRemoteMediaBuffer(...)` nur, wenn das Plugin Rohbytes für Inspektion, Transformationen, Entschlüsselung oder erneutes Hochladen benötigt. `fetchRemoteMedia(...)` bleibt ein veralteter Kompatibilitätsalias für `readRemoteMediaBuffer(...)`.

    `api.runtime.channel.mentions` ist die gemeinsame Schnittstelle für Richtlinien eingehender Erwähnungen für gebündelte Kanal-Plugins, die Runtime-Injektion verwenden:

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

    Verfügbare Hilfsfunktionen für Erwähnungen:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    Verwenden Sie für Erwähnungsentscheidungen den normalisierten Pfad `{ facts, policy }`.

    Mehrere Felder unter `reply`, `session` und `inbound` enthalten feldspezifische `@deprecated`-Hinweise, die auf den aktuellen Kernel für Kanalinteraktionen oder Adapter für ausgehende Kanalnachrichten verweisen; prüfen Sie die Inline-JSDoc der jeweiligen Hilfsfunktion, bevor Sie neuen Code darauf aufbauen.

  </Accordion>
</AccordionGroup>

## Speichern von Runtime-Referenzen

Verwenden Sie `createPluginRuntimeStore`, um die Runtime-Referenz für die Verwendung außerhalb des Callbacks `register` zu speichern:

<Steps>
  <Step title="Speicher erstellen">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="Mit dem Einstiegspunkt verbinden">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="Aus anderen Dateien zugreifen">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // löst einen Fehler aus, wenn nicht initialisiert
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // gibt null zurück, wenn nicht initialisiert
    }
    ```

  </Step>
</Steps>

<Note>
Bevorzugen Sie `pluginId` für die Identität des Runtime-Speichers. Die Form `key` auf niedrigerer Ebene ist für seltene Fälle vorgesehen, in denen ein Plugin absichtlich mehr als einen Runtime-Slot benötigt.
</Note>

## Weitere Felder auf oberster Ebene unter `api`

Neben `api.runtime` stellt das API-Objekt außerdem Folgendes bereit:

<ParamField path="api.id" type="string">
  Plugin-ID.
</ParamField>
<ParamField path="api.name" type="string">
  Anzeigename des Plugins.
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  Aktueller Konfigurations-Snapshot (sofern verfügbar, der aktive In-Memory-Laufzeit-Snapshot).
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  Pluginspezifische Konfiguration aus `plugins.entries.<id>.config`.
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  Bereichsgebundener Logger (`debug`, `info`, `warn`, `error`).
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  Aktueller Lademodus: `"full"` (Live-Aktivierung), `"discovery"` / `"tool-discovery"` (schreibgeschützte Funktionserkennung), `"setup-only"` (leichtgewichtiger Einrichtungseinstieg), `"setup-runtime"` (Einrichtungsablauf, der auch den Laufzeit-Kanaleinstieg benötigt) oder `"cli-metadata"` (Erfassung von CLI-Befehlsmetadaten).
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Einen Pfad relativ zum Plugin-Stammverzeichnis auflösen.
</ParamField>

## Verwandte Themen

- [Plugin-Interna](/de/plugins/architecture) — Funktionsmodell und Registry
- [SDK-Einstiegspunkte](/de/plugins/sdk-entrypoints) — Optionen für `definePluginEntry`
- [SDK-Überblick](/de/plugins/sdk-overview) — Subpfadreferenz
